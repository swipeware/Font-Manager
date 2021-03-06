Font-Manager
============

Bitmap Font Manager for Corona / Lua - provides font management, bitmap string management, animation and special effects.

This is v2. It has been completely re-engineered from v1, though it is pretty much the same (see end). It is a much more robust and coherent design than v1
which suffered a bit from being the first serious lump of Corona code I have written.

Allows the use of GlyphDesigner fonts - can be done either using an OOP style - e.g. BitmapString:new(), or using a display.newBitmapText() method which is 
fairly close to display.newText (these strings are multiline)

The main differences are that

(i) assigning text,anchorX,anchorY etc. won't work. The methods have similar functionality though, anchors, scales, positioning should work the same. You can use the
show() method which will copy text, anchorX and anchorY into the objec.

(ii) the display object is a display object (a group) but it is also a mixin (decorated with the bitmap functionality) so it cannot be directly used as class. The
best way to create a new class is to create a bitmap string and decorate it.

So for example, you can write code like:

	local str4 = display.newBitmapText("Hello World !",160,240,"retrofont",40) 

	transition.to(str4, { time = 1000, xScale = 2,yScale = 2})

notice the str4:getView() has been removed. However, it does still work - it is a dummy now whose only purpose is to make code that used getView() work unchanged.

You can also create strings like this in an OOP manner.

	local str4 = fm.BitmapString:new("retrofont",40):moveTo(160,240)

... same thing. the display.newBitmapText() is actually a shorthand to this to help people who aren't OOP minded. Which is fine :)

The font height (40 in this example) is the vertical total height of the font in pixels, including ascenders and descenders. So you can guarantee that a string created with a 40 pixel height will fit in a 40 pixel gap (unless you use multi-lines, obviously !). 

If you use the DEFAULT_SIZE constant, which is in both a string instance and the BitmapString class as a static, it will use the actual physical size of the font - the 1x font, not the 4x font. It will be consistent on differing resolution devices. If you actually *want* to use the 4x font (say) use font@4x as the font name, 
with DEFAULT_SIZE.

The font names check for scaling. So if you ask for a font called "retrofont" for an iPad Retina it will look for retrofont@4x.fnt, then it will look for retrofont@2x.fnt, then it will look for retrofont.fnt. A normal iPad will look for retrofont@2x.fnt, then retrofont.fnt etc. This is for compatibility with bmGlyph which can output fonts at different scales to save on texture memory in the device. This is completely independent of scaling in config.lua ; it works out the desired scale using the recommended code in the Corona article on Retina displays and looks for it manually.

The strings have their own internal direction and position, they also have modifiers so you can tweak the shape and size and rotation of individual
characters, either statically, or they can be animated automatically. 

The basic idea is that the string has a static, boring, non-animated position where it just 'is' - no modifier, no animation. From the point of view of the graphics system, this is where the string 'is' (for anchoring and so on) irrespective of how you animate or modify it. This is because if you anchored on the actual displayed graphic, the anchor points might move around in an arbitrary way. The exception to this is the tap/touch event listeners which operate on where the font characters actually are visually.

So to curve this string can be as simple as :

	str4:setModifier("curve")

and to animate this curve is actually easier :

	str4:animate()

and to stop it

	str4:stop()

There are stock animations (run main.lua in the Corona simulator !) and you can also make your own up, and customise the standard ones.  The stock animations (currently) are wobble, scale, curve, iscale, icurve, jagged, zoomin and zoomout. There are demos of these on the FontDemo app at my github (both animated and static.)

The pulse effect (see main.lua) is done by this code - this is a 'programmed modifier'

	function pulser(modifier, cPos, info)
		local w = math.floor(info.elapsed/360) % info.length + 1 									-- every 360ms change character, creates a number 1 .. length
		if index == w then  																		-- are we scaling this character
			local newScale = 1 + (info.elapsed % 360) / 360 										-- calculate the scale zoom
			modifier.xScale,modifier.yScale = newScale,newScale 									-- scale it up
		end
	end

and to use this instead, just

	str4:setModifier(pulser)

You can get rid of them individually, effectively.

	str4:setText("")

will vanish it, though it is still there, you can kill it as an object with :

	str4:removeSelf()
	
You can chain things if you want and not bother with references

	display.newBitmapText("Hello World !",160,240,"retrofont",40):setModifier("curve"):animate()

will create it, set its modifier and run it indefinitely.

Event listeners now operate as they do on any other viewgroup, this class does not have its own addEventListener methods.

You can set the font encoding with

	fm.FontManager:setEncoding("utf8")

currently unicode, utf-8 and utf8 are supported. UTF-8 format is supported up to six bytes

Colour tinting can be done using setTintColor() - colours everything, can be in-string and also can be modified.  Colour tinting looks like :

	"Hello{blue}blue {1,0,1}purple {}world"	

The character pair used to detect tinting instructions can be set using fm.FontManager:setTintBrackets("@(",")@") for example - defaults to {}  - colours are black, red, green, yellow, blue, magenta, cyan, white, grey, orange, brown. The delimiters must be standard ASCII irrespective of encoding used. These are global for an application ; don't have different delimiters for different strings at the same time.

You can also use a similar syntax to include any image in the string as {$crab} - the $ sign indicates it is an external non-font image (thanks to Richard9 for this idea) this currently loads icons/<name>.png but this will be changeable. The graphic will be scaled along with all the other letters, wobble, curve, animate like any other as well.

If you are relying on Composer or Storyboard to clean up your display objects, this will work but *only* if the animation is off. When the animation is on a reference to the object is maintained, so it will not garbage collect. You can use str4:stop() to stop animation or str4:removeSelf() to stop everything, and clean up the string. It will look like it works, but it doesn't, it will keep a reference.

Note: fm, assumes you've done something like fm = require("fontmanager")

Paul Robson.

Changes from v1
===============

a) setScale() no longer functions. Adjust font size to suit, or scale overall.

b) there is no FontManager object, really, though setEncoding(),setTintBrackets() and setAnimationRate() still work as there is a 'pretend' FontManager. These are now all
methods of BitmapString, though all affect the global state of the fonts, so setAnimationRate() sets the rate for all the bitmap strings, not just one.

c) curve and scale have been switched so they are the right way round. Previously they were 'visual' opposites.

d) FontManager:Clear() does not exist. The primary reason for this is that it maintained a reference to the object. If there is sufficient demand I will add a tracking of
creation on demand approach which will do the same thing.

e) You cannot directly subclass BitMapString, because it is now a mixin.

Your new method for a subclass should look something like

local function SubclassBitmapString:new(font,fontSize) 
	local newInstance = BitmapString:new(font,fontSize)
	.. do class specific initialisation.

	.. create mixin - you can do this with a for loop as well.
	newInstance.someFunction = SubclassBitmapString.someFunction

	return newInstance
end

f) Curve is now a static class in its own right rather than being a method of FontManage.

Kerning
=======

Thanks to Ingemar, the library now supports kerning - if it isn't present it just behaves as before. Note that this is turned on by default, so if you rely on spacing too precisely it could be slightly out. This can be turned on or off with the applyKerning(<boolean>) method, which turns it on or off according tot he boolean parameter.

Updates
=======

1/6/14 		UTF-8 now supports up to six byte characters
1/6/14		Autoscanning for @nx fonts.
1/6/14 		Padding in .fnt file is read and used to expand character box appropriately.
1/6/14		Added DEFAULT_SIZE (uses the actual physical bitmap size for 1x)
5/7/14 		Added in Ingemar's changes to use Corona's scaling system. 
6/7/14 		Couple of minor consequential bugs fixed from those changes.
6/8/14 		Kerning added.
Paul Robson 
paul@robsons.org.uk

Thanks go to : Richard 9 and Ingemar Bergmark on the forums, Sephane Queraud (BmGlyph) and Michael Daley (GlyphDesigner) for their assistance and ideas.

bmGlyph:			http://www.bmglyph.com
Glyph Designer:		http://71squared.com/glyphdesigner
