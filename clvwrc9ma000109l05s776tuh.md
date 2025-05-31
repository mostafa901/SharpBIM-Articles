---
title: "Outer CableTray radius"
datePublished: Tue May 07 2024 19:02:18 GMT+0000 (Coordinated Universal Time)
cuid: clvwrc9ma000109l05s776tuh
slug: outer-cabletray-radius
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1715612719317/09b7f864-347a-4429-bc73-85aa14c37cf9.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1715612732445/553539b0-b421-45c5-a91f-57b59a440010.png
tags: offset, radius, outerradius, mepcurve, cabletray, outer-radius

---

MEP fittings in Revit are families designed for MEP projects. Since these families can come in various forms, tracking values like the bend radius between two connectors can be challenging.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715107202744/e200119b-73f4-40cc-ad5e-4e511a7bdf59.png align="center")

One approach is to add a reporting parameter for the outer radius within the family, which can then be used for scheduling. However, manually adding this report to each family can be time-consuming.

Another possible method involves coding, which, although not straightforward, can be applied to all families once implemented.

To start, we need to identify the two connectors for which we want to determine the outer radius. From these 2 connectors we extract the origin points. The points are crucial for finding the center point of the arc. Getting the center point can be reached by drawing a perpendicular line to the cable tray curve direction and projecting the second point onto this line, by which we can locate the center.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715107220331/c2dff25c-c10f-4bfc-a92c-f533816c8af5.png align="center")

Next, draw an arc connecting the two points with the radius. The radius can be measured by getting the distance between any of the connectors to the center point we evaluated.  This approach will establish the centerline connection between the two MEP curves. Finally, by using the [**CreateOffset**function wit](https://www.revitapidocs.com/2017/450217f3-c0b5-42af-3a05-376ae383d28a.htm)h a distance equal to half the cable tray width, we can achieve our desired outcome.

Below is a sample code that summarizes the process. You need to pay attention to the angles required to draw the arc. The start angle must be less than the end angle. One common observation in most MEP Fittings, especially the elbow type, is that the path used to sweep the profile consists of two lines and one arc. These two lines provide a smooth connection when used. Therefore, you also need to consider this aspect to obtain an accurate result.

Let us know your findings.

```csharp
// assuming you already have the two connectors.
// we get the curves lines from its owner.
var horizontalLine = ((LocationCurve)connectorA.Owner.Location).Curve as Line;
var verticalLine = ((LocationCurve)connectorB.Owner.Location).Curve as Line;

// catch the distance and location of the 2 connectors and the distance between
var originA = connectorA.Origin;
var originB = connectorB.Origin;
var diagonalDistance = originA.DistanceTo(originB) * 2;

// get the sketchplan used to draw such cable trays
var sketchPlan = SketchPlane.Create(Doc, ((MEPCurve)connectorA.Owner).ReferenceLevel.Id).GetPlane();

// now we need draw a perpendicular line to any of the mep curves.
var perDirection = horizontalLine.Direction.CrossProduct(sketchPlan.Normal);
var perpL1 = Line.CreateBound(originA - perDirection * diagonalDistance, originA + perDirection * diagonalDistance);

var perDirection2 = verticalLine.Direction.CrossProduct(sketchPlan.Normal);
var perpL2 = Line.CreateBound(originB - perDirection2 * diagonalDistance, originB + perDirection2 * diagonalDistance);

// then project the second point over this perpendicular curve to get the center point
perpL2.Intersect(perpL1, out IntersectionResultArray inters);
//perpL1.Project(originB).XYZPoint;
var centerPoint = inters.get_Item(0).XYZPoint;
 
// arc requires 2 angles start and end
double angleX = sketchPlan.XVec.AngleTo((originA - centerPoint).Normalize());
double angleY = sketchPlan.XVec.AngleTo((originB - centerPoint).Normalize());

if (angleX > angleY)
{
    (angleX,angleY) = (angleY,angleX);
}
// draw the arc <= this is in the zero position "sketchplan origin point"
Curve arc = Arc.Create(sketchPlan, originB.DistanceTo(centerPoint), angleX, angleY);

// move the arc to the center point
arc = arc.CreateTransformed(Transform.CreateTranslation(centerPoint));

// create the offset needed to get the outer Radius
arc = arc.CreateOffset(((MEPCurve)connectorA.Owner).Width / 2, sketchPlan.Normal);
```