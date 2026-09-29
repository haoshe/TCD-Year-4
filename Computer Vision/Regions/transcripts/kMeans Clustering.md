# k-Means Clustering

*Lecture transcript, Computer Vision: Regions*

## Introduction

Okay, this is the second session within region segmentation. In this session we're going to look at a technique called k-means clustering, which allows us to identify the significant colors which appear within an image or within a section of an image. In the previous session we looked at connected components analysis, and that was where we were explicitly separating a binary image into distinct regions to represent the objects within that image.

This one is slightly different because this is not looking at spatial connectivity, it's really looking at similarity within color space. And so we're trying to find a number of pixels which have similar colors. We're putting it in with region segmentation because it often gets considered in the same type of technique.

## Why significant colours are useful

So to give you a practical example, it's something I return to quite a lot, is the notion that if you were looking at a person who was walking in a scene, that you might want to look at that person and to describe their clothing in terms of what they're wearing on their top, what their skin color is, what their hair color is, and what they're wearing on the bottom. So if we can identify significant colors associated with those parts of the scene, then we can describe the person as they move through the scene.

In the example on the right, the little boy is segmented from the scene, so ignore all the white background. This person has been segmented out from the scene. There are potentially still 16.8 million colors associated with the image on top. On the bottom, the number of colors has been reduced to four. So we've taken an image that has a significant number of colors, and we now can clearly describe within this image on the bottom that the boy is wearing a green t-shirt and has some sort of tan colored trousers on. That's really, really useful. And we can't do that with the image on top because there are too many different colors present, which are only very, very slightly different from each other. But it makes it infeasible for us to describe the colors on top, whereas this technique, K-means clustering, is going to allow us to do this.

What this means is that we can have very concise descriptions then of the colors that belong to an object or that belong within an image. And the idea can be used in order to facilitate object tracking. So we can use the colors on a person walking through the scene, and we can therefore track that this is the same person or is likely to be the same person as long as we keep a view of them.

The idea is particularly useful if you have people which are overlapping in the visual scene. So you have one person, say, walking in one direction and one in the other, and they overlap, so one of them is occluded. Hopefully, if they're wearing different colored clothes, we can identify which way each person went. Now we could be deceived, but the odds of that happening are relatively low.

We can use this technique to reduce the number of colors in the image, so we've seen that. But we can actually use it as a way of doing compression, if we so wish.

## What k-means clustering does

So the question becomes then, how do we find the best colors? How do we find the ones that are going to be representative for us? And we're going to use this technique called k-means clustering. So k-means clustering is going to create k clusters of pixels. Now the pixels don't have anything to do with each other. They're effectively k colors. So we're going to find k colors, and all of the pixels are going to be associated with one of those k colors. And therefore, each of those k colors has a cluster of pixels around it, which belong or are associated with that color.

We use an unsupervised learning technique. So we will look at learning within the course. It's quite a significant thing now within vision. And clearly, machine learning is taking off within the vision area.

## The basic algorithm

So let's have a look at the algorithm to do k-means clustering. In the basic algorithm, which is quite old, the number of clusters must be known in advance. If we don't know that in advance, then what we may do is we may try different values of k, and we may pick the one which gives us the greatest confidence at the end of the day. There's an awful lot that we can do in terms of post-processing this in order to see what sort of confidence we have in the segmentation which is created. And we can actually adjust the algorithm accordingly, or we can do post-processing to improve the results. I'm only going to look at the most basic version, the original version, which is still quite powerful in terms of what it does.

So what we do is we decide, okay, we're going to have k clusters, so we need k colors. And we start off with what are called k exemplars. Okay, so the exemplars are effectively sample colors. And in the original version of the algorithm, these k clusters, these exemplars, were set randomly. Okay, so this is non-deterministic, so it will operate slightly differently every time you run it.

There are other ways you could do this. You could extract possible exemplars from the image. So it talks about using the first k patterns, which would mean a pattern effectively is a color. So RGB value or whatever it happens to be, whatever domain you're working in. Or we can imagine randomly selecting pixels from within the image as our exemplars, if that's what we feel is appropriate.

In the first pass, what we do is we allocate the pattern, so these are the pixel values, each of the pixel values is considered a pattern. So we go through pixel by pixel, and we allocate the pattern of the pixel, so the RGB value or whatever it is, to the closest existing exemplar. So there's k exemplars, let's assume that's 10, or 20, or 100, but let's assume there's 10 exemplars. Okay, so we look at the 10 exemplars, we compare them to our current pixel, and whichever exemplar is closest, we allocate that pixel, that pattern associated with that pixel, to the cluster exemplar, to that cluster exemplar. And then we recompute the exemplar as the center of gravity of all the patterns which are associated to it.

In the second pass, having gone through the entire image in the first pass, we use the exemplars which are computed and we reallocate the patterns to the closest exemplars yet again. Okay, and we can have some redistribution.

## A one-dimensional example

So let's have a look at a simple one-dimensional example. So it's a little hard to do this in two dimensions. So what you're seeing here is, effectively this is over time, so we're going to allocate patterns to an initial set of exemplars. So I've got three exemplars here at the very start, so these are the red, the blue, and the green exemplars. Okay, and they don't mean any, the colours don't mean anything, it's only in order to give us something to refer to.

When the first pattern, or the first pixel value, is considered, it will give us, in this case, all we're looking at is one dimension. So let's assume this is grayscale. So it's giving us a grayscale of around 20. And the closest exemplar to that is the red one. So what we do is we allocate this to the red exemplar, and then we recompute the location of the red exemplar based on all the patterns that are associated with the red. In this case, there's only one pattern associated with it, so it becomes, it just moves to that pattern, to that location, to that grayscale.

In the next sample that's given, the next pattern that's given to us, it is also very close to the red exemplar. Okay, and so it also gets a red label. And then the exemplar is computed as the average of all the patterns associated with it. So it becomes the centre of those two patterns.

In our next line, we have a new pattern, which is just very slightly closer to the green exemplar than it is to the blue exemplar. So it becomes a green pixel, or a green pattern. I've got to keep using the word pattern. The next pattern appears slightly to the right, and it's again closest to the green. So the green exemplar becomes the average of those two. The next pattern appears even further to the right, and so the average of the green shifts even further over to the right. The next pattern, which appears, is very slightly closer to the blue exemplar, and so this becomes a blue pattern. The next one that appears is closer to the green, so it again draws the green exemplar further over to the right. The next one, which appears, despite being very close to the blue, to this green pattern, is closer to the blue exemplar, so it gets associated with the blue. And then, that's our last pattern that's been associated.

This line J is the second pass, where we recompute which exemplar each of the patterns is associated with based on proximity to the pattern. And you can see that this one, which effectively had been mislabeled at the start, gets reallocated in as a blue pattern instead of as a green pattern.

So again, let me say that the colours don't matter here. I'm just using colours effectively as labels in this particular case. And what you can see is that we've got now three separate clusters of patterns. These patterns could come from anywhere within the image. So if we were dealing with a greyscale image here, we've got some points which are very dark, we've got some points which are almost white, and we've got some points which are greyish. So this is how we would do our k-means clustering in a greyscale image.

## Applying it to a colour image

Clearly, we would like to do this in a colour image. So let's have a look at how that operates. So here, what we've got is our snooker image. And we've applied k-means clustering with 10, 15, and 20 random exemplars.

So the first thing that you'll notice, so let's just look at number 10 here. So this is on the left-hand side. So there, the k is the value 10. And just down here below, we have seven colours which are indicated. These seven colours are the colours that appear within this image. So despite the fact that we created 10 different exemplars, only seven of those exemplars ended up with patterns associated with them. So three of the exemplars effectively got no points associated with them and therefore got thrown away. It just depends on where those patterns or where those exemplars were put in the first place. And where the distribution of colours was within the image.

So, if we look at this, this is not the ideal representation of this image. That ball there was actually a little bit blue. So that's not very good. And that ball there is the white ball. So we're not very happy with that.

If we increase the number of exemplars to 15 or to 20, we start to get more and more accurate representations. The white ball back here, unfortunately, is still associated with some sort of green. And you can see that there are artificial contours appearing. This is the same as the effects you saw from quantisation. So as you reduce the number of possible colours or greyscales within a scene, you get what appear to be artificial contours. The same is going to happen in this domain because we're reducing the number of levels of green which are allowed within this scene. And so we effectively introduce these artificial contours into our scene.

The scene on the right, though, with the 20 exemplars, looks pretty good. It's only got 16 actual colours and yet we can understand the content of this scene reasonably well. So typically, more exemplars gives a more faithful representation. It will not be completely faithful. Well, I suppose if you put in an exemplar for every possible colour, then you'll have it being completely faithful. This is true.

So what I'm going to do is have a look at this example here, where we change the number of, the number k, so the number of exemplars that we start with each time. And as we increase the number of exemplars, the scene should become more and more recognisable as the original image. So we're all the way up here at k equals 20. And k equals 20 here is very, very similar, certainly to my eye, it's very similar to the original image. There are certainly some differences present, but most of the content of the image is possible for me to understand in this particular image. So I think that's quite a good output, that's quite a good outcome from the scene.

## Choosing the number of clusters

So back to k-means clustering again. We would like to choose the best number of exemplars. So how do we pick which one of these is the right answer, so to speak? We don't want too many. We're trying to reduce the number of colours that we consider within the scene.

One possible metric that was proposed was proposed by Davies and Bouldin to measure cluster separation. And so here what we have is k is the number of clusters. And for each cluster, we consider all other clusters. And we look for the maximum of the following metric. So we have two values sigma, and we have one value delta. So delta is the distance between the cluster centres. So how far apart are these two different clusters? And the sigma values measure what the average distance is from the centre of the cluster. So if we look at the maximum of that, then that looks for the two colours which are closest together in effect. And therefore the two that are worst, that are too easily confused. So the maximum, if we sum up all those maximum, that gives us a measure of cluster separation. And the lower the value the better.

What we found when we actually did some experimentation with this was that it doesn't work particularly well if the clusters are of very different sizes. So for example, in the image that you have above, there's a very small white cluster on the right hand side image, on the k equals 20 image. And there's a very small white cluster. There's a very small cluster associated with skin, with the red balls. There are very large clusters associated with green. And the DB index doesn't work particularly well where the cluster sizes are radically different.

There are lots of other things you can do. So you could check for each of the distributions. One of the things you could do is you could check to see if the distributions are normal. So if you have two clusters that are very close together, or if you have a cluster that's representing more than one colour, the shape of the distribution will not be anything like normal. And so you might take a distribution and split it into two. Or you might take two distributions which are very close together and shouldn't have been separated and join those two into one, where you get a single normal distribution as a result. So there are other ways in which we can do this, in which we can ensure that the segmentation we're getting, the separation between clusters is as good as possible.

## Non-determinism

And the final thing to look at from an example point of view, from a theory point of view, is that this is a non-deterministic algorithm. And if we run the algorithm over and over again, we get different exemplars each time. So here we've used 30 random exemplars each time. And you can see three radically different results. The images produced are not massively different. But the separation is certainly different within the image space. They're both, they're all, all three are relatively good representations. So this is a good technique. But you need to bear in mind that it's non-deterministic if you've used random exemplars.

## k-means in OpenCV

Then the last thing we have to do is we just need to look at the code that's used within OpenCV to produce k-means clustering. It's a little bit odd because this is a technique which is not intended to process images. And so it needs to be passed a one-dimensional array of samples. So if you look up here at the very top, we're creating a one-dimensional array of samples to store all the image data in. And we're effectively copying over the image data into the sample data.

To apply k-means is very straightforward. We need to get a certain number of labels and we need to know where the centers of those labels are at the end of the day. We need to pass in k and we need to pass in the samples. And then at the end, in order to create an image which has distinct colors in it, what we do is we put the centers of the colors. So the centers represent the average color, if you like, for the particular cluster. And all we have to do is we take the label that exists for each of the samples and we put it back in to the image location in the result image. Okay, and we have to do this for each channel.

## Summary

Okay, so that is the end of k-means clustering. It's a little bit different in terms of region segmentation. It is a segmentation technique and it often produces things that look like regions to us. But you need to bear in mind that k-means clustering is not taking any account of the spatial relationship between pixels. So if I jumble up the pixels and give you a random image, you'll get the same result as you would have got if you had given it the original image. So spatial position has nothing to do with it. And when we start doing processing after that, you need to bear that in mind.

When we look at some of the other techniques like watershed segmentation later on, you'll see that we do take into account spatial location as well. So there are techniques that will allow us to look at space as well as color simultaneously. Thank you for listening.
