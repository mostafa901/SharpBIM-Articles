---
title: "Rotate BoundingBox"
datePublished: Fri May 03 2024 20:05:53 GMT+0000 (Coordinated Universal Time)
cuid: clvr3umlx00040aii2vwo35m6
slug: rotate-boundingbox
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1715613653271/bf0e5023-2058-488c-a5c7-ba1a2dca6687.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1715613668227/6c1f2211-348b-4608-a59f-1bd82557ee2c.png
tags: revitapi, boundingbox, rotate-boundingbox, rotation-transfrom

---

BoundingBox, as the name suggests, is a box that contains elements inside it. The RevitAPI allows developers to access this box for various purposes like filtering, determining its position, dimensions, and more. It's important to note that the bounding box is always aligned with the cardinal axis. This means it doesn't change based on how the elements inside are rotated or scaled.

In the illustration below, the red dashed rectangle defines the bounding box around elements.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1714397279369/15b995b9-148e-4783-95ba-8a473b2ec984.png align="center")

This might be a bit confusing for new developers because one might initially think the box would also rotate. However, we can virtually rotate the bounding box using a Transformation. For instance, if we want a rotated transform, we need to define an angle and position. Think of it like drawing a circle with a compass where the needle is the position and the angle is how we rotate the head.

We can then obtain any point using `Transform.OfPoint(XYZ)`. So, rotating is figured out. The issue now is that if we rotate the bounding box, it may end up larger than it should be. For example, in the illustration below, the bounding box that originally fit a wall may no longer match the wall.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1714418363318/aa643e5c-c53a-40f2-884a-2eda6bffb708.png align="center")

To solve this problem, we need to determine the angle of the wall that best aligns horizontally or vertically. Then, after temporarily rotating the wall with the negative of the angle, getting the bounding box, rolling back the transaction, and finally rotating the bounding box, you'll achieve the desired result.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1714418553965/94f7b79d-e014-4a22-90c4-cadeed493710.png align="center")

I have created a pseudocode below to illustrate how this process works. This pseudocode generates detail lines of the bottom face of the rotated bounding box.

```csharp
// get the angle by which defines the element rotation
var angle = element.Direction.AngleTo(XYZ.BasisX);

// start a transaction
Trans.Start();
// rotate element
ElementTransformUtils.RotateElement(
	element.Document,
	element.Id,
	Line.CreateBound(bound.GetCG(), bound.GetCG() + XYZ.BasisZ),
	-angle
);
// regenerate the document to refresh the changes
RvtApi.Doc.Regenerate();
// extract the bounding box
bound = element.GetBoundingBox();
// roll back the transaction since we don't need to rotate the actual element
Trans.RollBack();

// create a rotation transform 
var transform = Transform.CreateRotationAtPoint(XYZ.BasisZ, angle, bound.GetCG());
// extract the points needed
var points = bound
	.Midpoints()
	.BottomCorners.Select(o => transform.OfPoint(o))
	.ToList();

// draw lines on Revit to illustrate the result
Trans.Start();
for (int i = 1; i < points.Count; i++)
{
	Doc.Create.NewDetailCurve(
		UiDoc.ActiveGraphicalView,
		Line.CreateBound(points[i - 1], points[i])
	);
}

Doc.Create.NewDetailCurve(
	UiDoc.ActiveGraphicalView,
	Line.CreateBound(points.Last(), points.First())
);

Trans.Commit();
```