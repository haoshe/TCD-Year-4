# Region Features

*Lecture transcript, Computer Vision: Regions*

## Recap

This is the second part of the lectures on regions. So in the last session we looked at connectivity of pixels, so specifically looking at 4 connectedness and 8 connectedness. You do have to make a decision as to which pixels are connected to other pixels in order to allow you to move a little further forward and to connect those pixels together into specific regions. We finished off by looking at a really important technique called connected component analysis, which basically allows you to scan through an image in two passes and to create a number of regions to describe the shapes within that image.

## Recognising the extracted regions

Now having extracted shapes from an image, so we've now effectively segmented some regions within the scene, we can go about trying to recognise those shapes that we've extracted. And the simplest way of doing that is to actually look at the features within the scene. So to look at the features of the regions that we've actually extracted.

## Area

So we're going to move on and the simplest type of feature that we can look at is actually the area. So if we've got a region, we can basically count the number of pixels in that region and that will give us the area. Now, there's lots of problems with that. It really depends on how far away the camera is from the object. And there's a whole pile of things going on here which make that a very, very simplistic feature. But it might be useful if we could normalise it in some way with respect to other things that we see.

## Minimum bounding rectangle

We can go a little further and there's a couple of types of processing which are particularly useful. One of those is to extract the minimum bounding rectangle. So we look at our shape and we find out what the tightest fitting rectangle is that would fit around our shape, our region that we've extracted.

And the simple way to do that is to try every possible orientation. So let's just assume we try every degree and we try a rectangle which is, you know, straight on. It's horizontal and vertical. The sides of it are horizontal and vertical. We move it. We find out where it fits most closely to the shape and we measure the size of that rectangle. And then we turn the rectangle by a degree and we do the same thing. We find out how it best fits the shape that we have and we keep doing that. And having done that, we will find the minimum bounding rectangle. We just pick the one which gives us the best answer.

Once we've done that, we can start to extract features which will certainly seem a little bit more useful in the area. So we can look at the length to width ratio of the minimum bounding rectangle. And that gives us some idea of the type of shape that we're considering.

We can consider the rectangularity of it. So we can look at the area of the object, which we said might turn out to be useful, and divide that by the length times the width, which is effectively the area of the minimum bounding rectangle. And if you have a perfect rectangle, then the area will equal the length times the width. So you'll get a one. And if you, well, you'll never get a zero, but you can get pretty close, depending on what sort of shape you have. It'd have to be something, like we say, not very complete, something with like an X, if you put an X into an image, which is very thin.

We can compute the convex hull to the minimum bounding rectangle area ratio. Now we haven't looked at the convex hull yet, so we're going to look at that in a little while. So we look at the area inside the convex hull, and we compare that to the area in the minimum bounding rectangle. So we will come to that in a little while.

There is support for this, as with most things to do with features within OpenCV. So you can just make a call to minimum area rect for a particular contour. Now a contour in OpenCV language is effectively a region. And so they describe the regions that you find using connected component analysis using a contour, which goes around the outside of that region. And it is a bit of terminology that you really need to try and work your way around. Contours in theory in vision mean edges. They don't really mean regions. So you're better off to explicitly talk about regions when you're discussing regions, rather than talking about contours, which is the terminology used within OpenCV.

## Elongatedness

We might want to know how long the object is, or how elongated the object is. Unfortunately, we can't use the ratio of the length and width of the minimum bounding rectangle, because we can have shapes like the green shape on the right hand side here, which can be quite long. So the elongatedness can be quite long.

By the looks of things here, the computation is poorer in terms of the measure of elongatedness. It shouldn't be coming up with the same number as the rectangle. So whenever you're programming, you should do a sanity check. And that sanity check should show you whether the values you're computing are correct or not. And there's definitely something wrong with this.

And so if you look down on the left hand side, you look at the circle, the square and the rectangle. So they're the perfect examples we've been given. And we're getting an elongatedness value of 1.85 for the rectangle, which seems fine. But then when you go over to the right hand side and you look at the unknown shapes, if you like, the ones that we're trying to classify, you'll see that we've got 1.82 for the red one, 1.81 for the bluish one, which is fine. They're fitting pretty well to what you'd expect for a rectangle. And then the third one is giving 1.82 for the elongatedness. So I have to say that's wrong. There's something going wrong in the way elongatedness is being computed for that. So you have to sanity check everything you do. And clearly I didn't do enough sanity checking of this particular example.

So the way elongatedness should be computed is by taking a ratio of the region area and dividing it by the square of the thickness of the object concerned. And in the first two cases, the red and the blue one, it's pretty clear that these are thick objects. Whereas the green one is very thin, so you expect a very different value of elongatedness for the green one in comparison to the two rectangles.

The formula that should have been used was the area of the object divided by 2d squared. Now, what d is, is the number of erosions that are required to make the object disappear completely. So we just take our region, our shape, and we apply erosion to it, and we wait until all the pixels disappear. And when all the pixels disappear, the number of iterations is the value d. We multiply it by 2 to give us the width, because erosion erodes from both sides, and we square that and divide that number or divide area by that number.

## Convex hull

The convex hull, which we talked about, so we looked at the idea of taking the ratio of the minimum bounding rectangle and the convex hull. So the convex hull is generally smaller than the minimum bounding rectangle, and there's a simple way of doing the computation, which is sort of illustrated by this diagram.

So let me just bring it up anyway, the algorithm. So we start with the top left point. So we're basically going to go down through the image, and we're going to find the top left point in the object. No other point will be above that. So we extend that vector out. So if you look at the second diagram where there's a dashed line, and then you'll see that there's an angle marked down to the next vector, and that's leading to the next point in the convex hull.

So what we're actually going to do is we're going to have a look at all the boundary points in the object, and we're going to find the boundary, the next boundary point, which minimizes that angle. So all the other angles of the vector from that top blue point to any other point on the boundary of the object will give a greater angle with respect to that dotted line. And so the one that has the lowest angle is the next point in the convex hull. And we basically just repeat that as we go around until we get back on the very right-hand side to the final point, so to the start point.

So the convex hull is easy enough to compute. Again, OpenCV will do it for you. So you don't even have to worry about implementing these algorithms. They're there for you.

## Moments and moment invariants

There are a huge number of features out there which cannot be easily explained. And I'd say most of the engineers here will be very comfortable with the idea of moments and moment invariants. They've probably come across them before. Computer scientists are less likely to have come across this concept. And it's basically a measure of the distribution of points within a shape.

So what you're looking at here, the moment xy, we're considering all points that are within a specific region. We're multiplying the ij of each point within the region. So we're looking at the location of the point times the value of, could be the gray level. It could be just binary data. So it could be a 1 or a 0 for f of ij. But we could use grayscale here if we wanted to. And so we could be looking at a distribution of the grayscale and the location of the points within the image. So the grayscale and the location within these moments.

Now, the problem with that, the first problem with that is that it's not centralized. That it's not based on the centroid of the shape that we're considering. So the first thing that we do with moments is we change those basic moments and we turn them into central moments. And so the central moments basically are computing a measure of distribution around the center of the object. And I'm not going to go through these formulas. In fact, there's another set of formulas which look even worse, right? I'll explain why they're important in a little while.

The other thing we might want to do is get rid of scale. So scale can create issues for us. So if we want to consider a shape, we might want to consider the number 9. We don't care whether it's a giant big number 9 or whether it's a small number 9. And so we can do scale invariance as well. So another change to the central moments.

And then if we really want to go further, what we can do is we can take those normalized, centralized moments and we can compute some moment invariants. And to be honest, other than the first one or two of these, you can't really come up with intuitive explanations for what these are. But they're descriptions of a shape which incorporate a measure of the distribution of the shape with respect to the centroid. And in a scale invariant way. And so we may find that one of these magic seven invariants might be useful in order to allow us to classify the shape.

So let me show you an example here. So if you look on the table on the right hand side, so we're looking at the 10 digits, 0 through 9. And we've computed a number of things like the perimeter, the area, the bounding box, the convex hull, all these sort of things. And what you can see then is you can see the Hu moments. And that's what this slide is here for, is just to show you these Hu moments.

So if we were to look at the first of those moments, so we've got values of 0.17 ranging up to a top of 0.47. So it's very clear that there's, it looks like about five of the shapes have values around 0.45. And then another five of the shapes have values that range from 0.17 to 0.23. So that particular moment is going to allow us to distinguish five of the numbers from five of the other numbers. It's not allowing us to distinguish individual numbers, but it's certainly helping. And then the other moments will give us other measures and potentially allow us to recognise the objects if we can just choose the right moments or the right features to use. Again, moments are provided for you automatically within OpenCV. So it's, it's very straightforward to use them.

## Concavities

So we've got a minimum bounding rectangle. We've got a convex hull. Having computed the convex hull, we might actually be interested in the concavities which are created by the convex hull. So in between all those points that we identified on the outside of the convex hull, it would be quite useful to know if there's any concavities.

So if we consider our numbers again, 0 through 9, it should be pretty clear to you that the 9 and the 6, for example, both have a single large concavity. So if you look at the table here on the left hand side and you go down to the 6, you'll see it's got a number of concavities. It's got three concavities. But if you look carefully on the right hand side of that, it's got 3.1 at 317. So 317 is degrees. 3.1 is a measure of area. And then the 0.3 at 94 degrees. So that doesn't matter. It's a very, very small number.

And if we go down to the 9, you'll see it has four concavities. I'll explain why they have so many concavities. But the first concavity is around the same size as the one in the 6. It's area 3 at 143 degrees. And then 0.1 is the next largest concavity. So it should be pretty clear to you that, you know, the 9 has a concavity at the bottom left. And the 6 has a concavity at the top right. And the large concavity that we're finding, the largest concavity for each of those two, corresponds pretty well with where we would expect to see a concavity for those two different shapes.

And the reason you get a large number of concavities even on the 9 and the 6 is that you're looking at pixel based representation. And so you get these tiny, tiny little concavities in between different pixels, just depending on how the quantization has happened within the image. And that's just nature of the beast.

So if you look down at the number of concavities, it clearly doesn't make a huge amount of sense. You know, we've got 0 for the shape 0. That's almost unusual. And then you go down and if you look at the number 7, it's got 9 concavities. And that's all got to do with that diagonal line. So you've got a step effect on the diagonal line. And it creates all these really, really tiny concavities when you're doing the processing, which can effectively be thrown away. So the concavities that we've analyzed here are just the only ones we're interested in are the ones with large concavities. But what large is not clear. So we have to be careful not to throw things away too soon.

## Example: recognising licence plate digits

So this example was supposed to be showing us how we might try and do recognition of these numbers specifically. The ones on the left then, so if you look down below the license plate, you'll see samples of the numbers from 0 to 9. Those samples then appear on the table down below 0 through to 9. And we've got the various ratios we talked about before, the height-width ratio, the convex hulls, the bounding box ratio. And then we have a number of holes and we have a number of concavities and the size of the concavities really matters here.

And then over on the right-hand side, what we've done is we've analyzed the shapes that appear in the license plate. So the D has been analyzed to be a 0. Not surprisingly, because we haven't taught our system anything about letters. And so D looks very like 0. So not surprisingly, I've come up with 95037825 for this particular license plate. And if you look at the numbers that are being computed, what you should see is that the numbers correspond very closely with the examples that we've given, the samples that were given for 0 through 9. Okay, so that's the type of recognition we might do with these type of features.

Now we're not dealing with recognition in this particular session. And we'll look at using these features in part of our deep learning lectures when we're looking at linear classifiers. And linear classifiers are actually typically used as the last stage of classification within a deep network. So they actually are really important that they appear in there. But we can use linear classifiers on their own if we just have features like this. And we can combine features together and use the linear classifier to do the recognition or the classification.

Okay, so again, this is easy enough to find. We can get our convexity defects, right? So we're getting our convex hull and we're getting the concavities or what they call the convexity defects from OpenCV.

## Perimeter length and circularity

Moving on from that then, we can look at things like the perimeter length. And so we could look at the length of the contour as in, remember, contours are regions. But in fact, they're representing the points around the region. And so we could just look at the number of points there. We should technically take into account the difference between a 45 degree and a vertical or horizontal. And we can compute things like circularity from this, which is very nice. So circularity is just using the area and the perimeter length that we just computed.

## Summary

Now, so that's the end of features of region based features. And what we've looked at, just to summarize, is we looked at area, which is a little bit too simplistic. But we would use the area then in combination with other features and we'd be allowed to do recognition on the basis that would be useful for recognition on that basis. And we looked at the minimum bounding rectangle. And then features that can be extracted from that minimum bounding rectangle. We looked at elongatedness. And you remember my example is actually wrong in elongatedness. So I'm doing something wrong in my computation there. But we should be eroding the individual shape, finding out how many erosions it takes to get rid of this shape completely. And then combining that with area to give us a measure of elongatedness. We looked at the convex hull. So the convex hull and the minimum bounding rectangle are often combined because the comparison of the two gives us some really interesting things. And that convex hull also allows us to extract concavities. And in fact, the representations of regions within OpenCV lets us extract the holes very easily. And the only thing I've skipped there are the moments and the moment invariants. So they're a little harder. They're not as intuitive to compute, but they can distinguish very reliably between different types of shapes. And so we will use those things.

But you can imagine, you can see the difficulties here. We've got this massive list of features. And how do we decide which features we're going to use? So that's really, really hard. It's very difficult for us to make a decision on that. And then the final thing we looked at was the perimeter length and associated with that a measure of the circularity of the shape that we're looking at.

So that's the end of our introduction to features as part of regions. Bear in mind that these are there really so that we can use them as a measure to use in recognition. So they're important when we look at things like linear classifiers. Thank you very much.
