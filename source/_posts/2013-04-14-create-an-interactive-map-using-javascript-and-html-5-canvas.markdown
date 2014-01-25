---
layout: post
title: "Create an interactive map using Javascript and HTML5 Canvas"
date: 2013-04-14 19:11
comments: true
published: true
categories: js
---


I will show you how to create a custom interactive map using the HTML5 `<canvas>` element.  
You can check the result: <a href="http://riskmap.filkor.org" target="_blank"><strong>DEMO here</strong></a>  
The map size on the demo is 1920x1080 pixels.

<a href="http://riskmap.filkor.org" target="_blank">
<img class="aligncenter" src="/images/risk/map-sample.png" alt="" title="Risk map">
</a> 

Creating simple interactive maps is not so hard, you can see examples [here](http://www.html5canvastutorials.com/labs/html5-canvas-world-map-svg-path-with-kineticjs/), or [here](http://raphaeljs.com/world/).  
But when you need more customization, like different territory textures, shadows, special font-styles...etc, well then you have to use different tools for this job.  

What you will need:

- Vector graphics editor to draw the base paths for the map (I used Illustrator)
- A general graphics editor like Photoshop, we will import the paths from Illustrator to here
- [KineticJS](http://kineticjs.com), a Javascript library, for the `canvas` manipulation
- A little Javacript knowledge 

<!--more-->

#### Draw the base paths with Illustrator

First we need to draw the paths for the map. Well, I can't draw. I got 
<a href="/images/risk/risk-original.jpg" target="_blank">this picture</a> from the web and used this as a template, when I was drawing the paths.  
Open up Illustrator and make sure you set the size correctly, then our job is simply to draw the paths with the pen tool, based on the existing map image.

The drawing process looks like this:  
<iframe width="300" height="169" src="http://www.youtube.com/embed/7dcYbQzhBhg" frameborder="0" allowfullscreen></iframe>

When you are done the groups will look like:  
 
<img class="aligncenter" src="/images/risk/risk-1.png" alt="">  

A few notes: when you are drawing, make sure that the territories are not overlapping each other. I like to draw the continents first, then draw the border lines separately, duplicate them and join these paths to a single territory (as you can see on the video above), so you are using the same paths for the neighbour territories, and we can avoid overlapping.  


####Save the paths as SVG and parse the path data

When you have drawn the paths you can save it as SVG:  
(File -> Save As...)
<img src="/images/risk/saveas.png" alt="" class="aligncenter">

Now you have a .svg file, let's call it `risk-paths.svg` which is plain XML by the way, and it's content looks like this:

<pre class="prettyprint lang-svg" ...>
&lt;svg xmlns=&quot;http://www.w3.org/2000/svg&quot;&gt;
	&lt;g id=&quot;Alaska&quot;&gt;
		&lt;path fill=&quot;none&quot; stroke=&quot;#010101&quot; stroke-width=&quot;0.75&quot; d=&quot;M141.484,115.597c3.763,0.172,7.757-0.111,11.33-1.1   c9.971-2.769,11.077,0,11.077,0s-1.106....&quot;
	&lt;/g&gt;
	&lt;g id=&quot;Ontario&quot;&gt;
		&lt;path fill=&quot;none&quot; stroke=&quot;#010101&quot; stroke-width=&quot;0.75&quot; d=&quot;M386.977,281.79v12.17l-2.492,4.57v36.557   c0,0-4.57,0-5.816,0c-1.246,0-3.739-4.985-3.739-4.9....&quot;
	&lt;/g&gt;
	&lt;g id=&quot;Alberta&quot;&gt;
		&lt;path fill=&quot;none&quot; stroke=&quot;#010101&quot; stroke-width=&quot;0.75&quot; d=&quot;M153.781,259.181   c4.012....&quot;
	&lt;/g&gt;
	...
	...
</pre>

What we need here are the `d="M153.781,259.181   c4.012...."` values inside the `<path>` elements.  
In order to read and manipulate the paths with Javascript we need to parse this XML code, and put the paths data inside a Javascript object like this:

<pre class="prettyprint">
//paths.js

var TerritoryPathData = {
	Alaska: { path: 'M141.484,115.597c3.763,0.172,7.757-0.111,11.33-1.1   c9.971-2.769,11.077,0,11.077,0s-1.106....'},
	Ontario: { path: 'M386.977,281.79v12.17l-2.492,4.57v36.557   c0,0-4.57,0-5.816,0c-1.246,0-3.739-4.985-3.739-4.9....'},
	Albera: { path: 'M153.781,259.181   c4.012....'}
	...
	...
}
</pre>

The easiest way to do this is by using an XML parser libaray. I used PHP's [SimepleXML](http://php.net/manual/en/book.simplexml.php) library for this job.  
An exaple code for the usage would be:

<pre class="prettyprint">
//parseSVG.php

&lt;?php
	$data = file_get_contents('risk-paths.svg');
	$svgElements = new SimpleXMLElement($data);

	foreach ($svgElements as $element) {
		$continents = 	array('Europe', 'Asia', 'NorthAmerica', 'Australia', 'SouthAmerica', 'Africa');
		if (!in_array($element['id'], $continents ))
			echo $element['id'] . ': { path: \'' . $element-&gt;path['d'] . '\'},&lt;br&gt;';
	}
?&gt;
</pre>

Note that it requires the `risk-paths.svg` file we created earlier.  This code skips the continents like 'Europe'.. we just need the single territories path data right now.
Open up this PHP file in your browser (on localhost) and you will see the parsed data. 

Now we have this object `TerritoryPathData` containing our parsed data. Put this inside a separate file (say `paths.js`).

We will include `paths.js` inside our index.html. KineticJS will read this data and build his inner `Kinetic.Path` objects based on the contents of `paths.js`, then we can start to play with interactivity through the KineticJS functions (hover events, etc...).  


####Setting up KineticJS, and draw our backgroud image

In the above section we took care of the territory paths. Draw them in Illustrator, parsed the SVG and saved the paths data into `paths.js`.   
Now we will use KineticJS to read this data into a Kinetic.Path object. Why we use KineticJS ? We can reduce our work with it. It supports high performance path detection, groups, layers, transitions and much more. We dont have to code as much as with the native canvas functions. Download it from the <a target="_blank" href="http://kineticjs.com">website</a> and include it to your index.html.

So let's get started with KineticJS:  
I created a file `risk.js`, and wrapped everything around a `Risk` object:

<pre class="prettyprint">
//risk.js 

var Risk = {

	/**
	 * Settings Object, holding application wide settings
	 */
	Settings :{
		globalScale: 1,
		colors: {yellow: '#ff0', green: '#0f0', blue: '#00f', red: '#f00', purple: '#f0f', cyan: '#00ffe4'},
	},

	/**
	 * Our main Territories object
	 * It looks like:
	 * Territories: {
	 *     Alaska: {path: Kinetic.Path Object, color: String, name: 'Alaska', ...},
	 *	   ... 
	 *	}
	 */
	Territories: {},

	/**
	 * Some other variables
	 */
	stage: null,
	mapLayer: null,
	topLayer: null,
	backgroundLayer: null,

	/**
	 * Functions that we will use
	 */
	init: function() { },

	setUpTerritoriesObj: function() { },

	drawBackgroundImg: function() {	},

	drawTerritories: function() { },

	divideTerritories: function() { }
}
</pre>

Before we start coding, the process is roughly the following:  

- `Risk.init()` will triggers the whole application, calling the other functions from there.     
- We have to populate the `Risk.Territories` object, this object contains essential informations about the territories, like actual color etc.. we will done this with the `setUpTerritoriesObj()` function.  
- Show the background image. We need to draw this image in Photoshop, I will show how to do this, immediately.  
- And finally, draw the actual territories with KineticJS, set up the basic events like mouseover..  
- `divideTerritories()` is just a 'helper' function, it looks good on the demo when the territories got different colors at the beginning.


The tricky part comes now. I used the following image as the background:

<img width="100%" src="/images/risk/map_grey.jpg" alt="">

How it's done? You can export your paths from Illustrator to Photoshop.  It will convert your groups into Photoshop layers. (Go to Illustrator->File->Export..->choose *.PSD as your format, uncheck Anti-alias)
Open up this .psd file and you will see your paths as actual Photoshop layers now. Pretty.  
With Photoshop you can style it as you want, for example you can see a colored one <a href="/images/risk/risk-colored-small.jpg" target="_blank">here</a>. Now we are using a grey map because the actual colored territories will be drawn with Javascript.  

I will not write down the actual styling process because the post would be too long. If you have some Photoshop skills, you can do it quite easily. (Go to "Blending Options.." I used a little outer- and inner glow - for the grey map too -, stroke on the continents, and color overlay)  

Now why 'exporting to Photoshop' is a tricky thing? Because you need to export it **perfectly**. Do not modify the paths after you saved as SVG (and parsed the path data), otherwise the grey background and the actual territories what we make with KineticJS will not fit properly.  
Also, make sure you export from Illustrator at exactly the same size as you saved the SVG (in our case it's 1920x1080).

So when we are done with the background (I will call it `map_grey.jpg`), we can complete our `drawBackgroundImg()` function above:

<pre class="prettyprint">
drawBackgroundImg: function() {
	Risk.backgroundLayer = new Kinetic.Layer({
		scale: Risk.Settings.globalScale
	});
	var imgObj = new Image();
	imgObj.src = 'img/map_grey.jpg';
	
	var img = new Kinetic.Image({
		image: imgObj,
	});
	Risk.backgroundLayer.add(img);
}
</pre>

As you can see we put the background image inside a `Kinetic.Layer`. You can think of this layer what you see in graphics editing programs. 
We will 'write' the territory paths and names to a different layer, too.   



####Draw the territories with KineticJS

Now we have our background, so we can draw the territories 'onto' this gray map. So let's write our functions which will draw the actual territories. Open up `risk.js` again:

<pre class="prettyprint">
init: function() {
	//Initiate, or populate our main Territories Object, it contains essential data about the territories current state
	Risk.setUpTerritoriesObj();

	//Initiate a Kinetic stage
	Risk.stage = new Kinetic.Stage({
		container: 'map',
		width: 1920,
		height: 1080
	});

	Risk.mapLayer = new Kinetic.Layer({
		scale: Risk.Settings.globalScale
	});

	Risk.topLayer = new Kinetic.Layer({
		scale: Risk.Settings.globalScale
	});

	Risk.drawBackgroundImg();
	Risk.drawTerritories();

	Risk.stage.add(Risk.backgroundLayer);
	Risk.stage.add(Risk.mapLayer);

	Risk.mapLayer.draw();

	Risk.divideTerritories();
},

/**
 * Initiate the  Risk.Territories Object, this will contain the essential informations about the territories 
 */
setUpTerritoriesObj: function() {
	for(id in TerritoryNames) {

		var pathObject = new Kinetic.Path({
			data: TerritoryPathData[id].path,
			id: id //set a unique id --> path.attrs.id
		});

		//Using a sprite image for territory names
		//see: drawImage() -- https://developer.mozilla.org/en-US/docs/Canvas_tutorial/Using_images , and see Kinetic.Image() docs for more
		var sprite = new Image();
		sprite.src = 'img/names.png';
		var territoryNameImg = new Kinetic.Image({
			image: sprite,
			x: FontDestinationCoords[id].x,
			y: FontDestinationCoords[id].y,
			width: FontSpriteCoords[id].sWidth, //'destiantion Width' 
			height: FontSpriteCoords[id].sHeight, //'destination Height'
			crop: [FontSpriteCoords[id].sx, FontSpriteCoords[id].sy, FontSpriteCoords[id].sWidth, FontSpriteCoords[id].sHeight]

		});

		Risk.Territories[id] = {
			name: TerritoryNames[id],
			path: pathObject,
			nameImg: territoryNameImg,
			color: null,
			armyNum: null
		};
	}
	
},

drawTerritories: function() {
	for (t in Risk.Territories) {
		
		var path = Risk.Territories[t].path;
		var nameImg = Risk.Territories[t].nameImg;
		var group = new Kinetic.Group();

		//We have to set up a group for proper mouseover on territories and sprite name images 
		group.add(path);
		group.add(nameImg);
		Risk.mapLayer.add(group);
	
		//Basic animations 
		//Wrap the 'path', 'group' and 't' variables inside a closure, and set up the mouseover / mouseout events for the demo
		//when you make a bigger application you should move this functionality out from here, and maybe put these 'actions' in a seperate function/'class'
		(function(path, t, group) {
			group.on('mouseover', function() {
				path.setFill('#eee');
				path.setOpacity(0.3);
				group.moveTo(Risk.topLayer);
				Risk.topLayer.drawScene();
			});

			group.on('mouseout', function() {
				path.setFill(Risk.Settings.colors[Risk.Territories[t].color]);
				path.setOpacity(0.4);
				group.moveTo(Risk.mapLayer);
				Risk.topLayer.draw();
			});

			group.on('click', function() {
				console.log(path.attrs.id);
				location.hash = path.attrs.id;
			});
		})(path, t, group);
	}				
}
</pre>

If you look back at the demo, you can see the territory names. The names were made with Photoshop.


We can't export all the names as images, we have to create a sprite from them. So one image will contain all the names (therefore the whole application will load faster), this sprite looks like this:

<img src="/images/risk/names.png" class="aligncenter" width="160px">

How did I make this sprite? Well, it's a long story, but I heavily used Photoshop scripts for this job.. I modified the layersToSprite.jsx script what you can find online. 

Then inside the `setUpTerritoriesObj()` function, every territorry will get their proper name image as we 'crop out' from
the sprite based on the data in `coordinates.js`. (This data also parsed with certain Photosop scripts I made)

Lastly, as you can see, I added some actions to the 'mouseover', 'onlick' event (changing the opacity of the territory). For this, I grouped together the actual territory and the territory name because it's easier to handle these events in KineticJS this way.


####Final conclusion

From now I suggest try to read the actual source code (with Chrome developer tools for example), because this post is getting really long :)

I have tried to cover how to make a simple custom interactive map, but as you can see ,even for this simple map, you need to get your hands dirty, and a lot of things to do..

I hope, though, that if you want to create a custom map with Javascript and HTML5 this post will help you to start.