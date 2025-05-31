---
title: "Section line location"
datePublished: Fri May 03 2024 19:53:33 GMT+0000 (Coordinated Universal Time)
cuid: clvr3errh00030al9heog5f4p
slug: section-line-location
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1715613899687/47413c57-062f-45f0-ac67-744aeeb23132.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1715613909088/2c41469d-57e4-45c9-b083-d5feaac2eb5c.png
tags: revitapi, sectionline, cropbox

---

Drawing a section over a plan drawing is crucial for getting a better view of how an element is positioned or constructed. For instance, we can easily observe and manage a wall's height through a section.

Assuming we already have a section view and need to draw the section line that shows its position on a plan, this is achievable using the Revit API, albeit with some limitations.

Let's begin with what the Revit API can offer. Firstly, we obtain the crop box of the section view `sectionView.CropBox`, then we acquire its `Transform`. This transform enables us to convert the bounding box corner points to world space coordinates. Through trial and error, I discovered that the maximum point of the Section view is the one located above the section line, while the minimum point indicates the depth of the section. Refer to the snapshot below illustrating a 3D Bounding box extracted from the section View.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1714333942168/bcb09e55-3abf-4c33-a080-6fd55771eff5.png align="center")

Since we already have the maximum point from the crop box, which is the Top Right corner, we can easily determine the Top Left corner. By using these two points, we can draw a line over the plan view. Refer to the code below to see how this works.

```csharp
var cropBox = sectionView.CropBox;
var topRight = cropBox.Max;
var topLeft = new XYZ(cropBox.Min.X, cropBox.Max.Y, cropBox.Max.Z);
topRight = cropBox.Transform.OfPoint(topRight);
topLeft = cropBox.Transform.OfPoint(topLeft);
var line = Line.CreateBound(topRight, topLeft);
// a transaction must be opened here
planView.Document.Create.NewDetailCurve(planView, line);
// commit transaction
```

### Limitations

* The method only represents the bounding box of the section view and does not consider the actual segment of the section line.
    
* If the section line is segmented or cut to show different depths, this will be disregarded.
    
* If the bounding box changes from the section line, the drawn line will align with the section view.
    

As of the date of publishing this article, access to the section line segments and the bubble is still limited in the Revit API. Fortunately, [this is already a requested feature](https://forums.autodesk.com/t5/revit-ideas/revit-api-adjust-section-line/idi-p/11251294) on Revit Autodesk Ideas.