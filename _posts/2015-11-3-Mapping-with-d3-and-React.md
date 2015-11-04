---
layout: post
title: Mapping with d3 and React
---
I've been using React more in recent weeks, and really love the improvement over spaghetti JQuery DOM Manipulation. This past week I started working on integrating d3 into React. 

I've always been interested in the South China Sea, and making a basic map / information about the dispute there seemed like a good project to test things out. For the anxious, you can skip to [the final product](http://southchinasea.co) or check out the code [here](http://github.com/caacree/South-China-Sea). 
 
Throughout the process, I relied heavily on [this post](https://medium.com/@sxywu/on-d3-react-and-a-little-bit-of-flux-88a226f328f3#.3x0r5vagi) by Shirley Wu, who deserves some kind of award for most elaborate/detailed example project. Anyone interested in d3 and React should check out that post/repo first. 

The most important consideration when integrating the d3 into React is keeping a consistent DOM. Using d3's to insert or remove items from the DOM will break React. I started out trying to use React for nearly everything, and gradually transitioned back to d3 on the parts that didn't work well. React lacks the speed of d3 for large transitions and updating, and d3's data interface is probably the biggest advantage to using d3 in the first place. React's strengths lie in tracking state and adding / removing elements from the DOM. 

One difference between maps and most example apps I've seen so far is the amount of data. My map has about 100 features with tens of thousands of data points. While React can handle that level of traffic for most purposes, features like map panning which update dozens of times a second will slow it down. I was initially re-rendering on each pan, which felt very laggy. Switching back to normal d3 zoom and panning, called during the ComponentDidUpdate method, kept things smooth. 

For similar reasons, I kept some style changes attached to the zoom method. Using React would re-render the entire svg if I wanted to keep the strokes proportionally thin after zooming, but with d3 it's a quick style update. Again, normally React is fast enough, but for large datasets there is a noticeable lag in re-rendering the entire area. 

Other than that, the main challenge was just getting the semantics down. Some tips on that:
* If you're going to use d3 manipulations on a component, set it to be a variable in ComponentDidMount and/or ComponentDidUpdate.  
 ```  
componentDidMount() {
	this.d3Node = d3.select(ReactDOM.findDOMNode(this));
	...
}
 ```
* Save important variables in the object since components will re-render on update. For example, I needed to save my current zoom specs (translate and scale) so that whenever the component was updated I could reinitialize the zoom correctly. 
* Use React for small updates, d3 for large ones. React really does make simpler code and I have no doubt will be easier to maintain, and is accordingly preferred when there's no penalty. I used React to update selections, mouseovers, and highlights, since those all only update one element at a time. 

And here's a couple other random notes from this project:
* If you're using Flux, don't do this:  
 ```
SelectionStore.dispatchToken = AppDispatcher.register(function(action){
	switch (action.actionType) {
		case Constants.CHANGE_SELECTION:
			setSelection(action.data);
		...
		default: 
			return true;
		SelectionStore.emitChange();
	};
 });
```
 This can/will call the change event before your function has finished performing. In this case, it emitted the change before `setSelection` had a chance to change the state. So components tried to update the state before it had changed, and nothing happened. Putting the emitChange at the end of `setSelection` and similar methods makes it clear when the function is called and what will be returned. 
* Some things never make sense. I wasted a day trying to set hard panning limits on the map consistent across zoom scale. Various examples found online fell short, and trying to decipher how d3 changes translate values on each zoom was an exercise in frustration. I'm guessing it's some aspect of the Mercator projection, but in the end I just resorted to an ugly hack on the pan limits that keeps them stable enough no one will notice.
		
	```
		let t = d3.event.translate,
		s = d3.event.scale,
		factors = [s*.4, 1.5-s*.5, s*1.1, 1.5-s*.7];
		t[0] = Math.min(width / 2 * (s-factors[0]), Math.max(width / 2 * (factors[3]-s), t[0]));
		t[1] = Math.min(height / 2 * (s - factors[2]) + 230 * s, Math.max(height / 2 * (factors[1] - s) - 230 * s, t[1]));
	```
Beautiful. 
