---
title: "Filtering Elements Visible in a Viewport"
seoTitle: "Visible Elements in Viewport: Filtering Guide"
seoDescription: "Identify visible elements in Autodesk Revit viewports by determining boundaries, converting to model space, and applying precise 3D bounding filters"
datePublished: Sat Mar 22 2025 05:43:34 GMT+0000 (Coordinated Universal Time)
cuid: cm8js9u2u000q09ju5i1x7c7y
slug: filtering-elements-visible-in-a-viewport
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1742621909976/7a997a1f-0405-4e8b-9278-99057e6815e8.png
tags: csharp, revitapi, viewports

---

## **Introduction**

When working with Autodesk Revit API, a common challenge is **identifying elements that are visible within a specific viewport** on a sheet. Unlike retrieving all elements in a view, filtering only those that appear within the viewport requires additional calculations.

In this article, we will discuss how to achieve this by:

* Determining the **viewport boundaries** on the sheet.
    
* Converting those boundaries to **model space** coordinates.
    
* Accounting for the **view range** and **depth clipping** settings.
    
* Constructing a **3D bounding filter** to precisely capture visible elements.
    

By following this approach, we can efficiently select **only the elements that are visible within a viewport**, avoiding unnecessary selections and improving automation workflows in Revit.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1742621494684/dba8f9b6-7379-4b13-b5c5-c5c6595a5b09.gif align="center")

Let's dive into the implementation details!

### **Steps to Retrieve Elements Visible in the Viewport:**

1. **Obtain the viewport outline** – The first step is to get the 2D outline of the viewport on the sheet.
    
    ```csharp
    var outline = viewPort.GetBoxOutline();
    ```
    
2. **Convert the viewport corners to model space** – Since the viewport exists in sheet space, we need to transform its min and max points into model space using the associated view.
    
    ```csharp
    var p0 = outline.MinimumPoint.ConvertToModelSpace(viewPort);
    var p1 = outline.MaximumPoint.ConvertToModelSpace(viewPort);
    ```
    
    The `ConvertToModelSpace` method is defined as follows: (the code is inspired from [RPTHOMAS108](https://forums.autodesk.com/t5/revit-api-forum/getting-element-coordiantes-on-sheet/m-p/9785396))
    
    ```csharp
    public static XYZ ConvertToModelSpace(this XYZ pointOnSheet, Viewport viewPort)
    {
        // Get the document associated with the viewport.
        var Doc = viewPort.Document;
    
        // Retrieve the model space view associated with the viewport.
        var modelSpaceView = (View)Doc.GetElement(viewPort.ViewId);
        
        // Retrieve the transformation matrix of the view’s crop box.
        var viewTransform = modelSpaceView.CropBox.Transform;
    
        // Compute the center point of the model space view outline.
        BoundingBoxUV modelSpaceOutline = modelSpaceView.Outline;
        var cgModelSpaceUV = (modelSpaceOutline.Min + modelSpaceOutline.Max) / 2;
        var cgModelSpace = new XYZ(cgModelSpaceUV.U, cgModelSpaceUV.V, 0);
    
        // Compute the center point of the viewport outline.
        Outline vportOutLine = viewPort.GetBoxOutline();
        var cgVportOutline = (vportOutLine.MinimumPoint + vportOutLine.MaximumPoint) / 2;
    
        // Compute the offset between viewport space and model space.
        var offset = cgVportOutline.Subtract(cgModelSpace);
    
        // Retrieve the view scale.
        int Scale = modelSpaceView.Scale;
    
        // Convert the sheet point to model space using the transformation and scale.
        var pointOnModel = viewTransform.OfPoint((pointOnSheet - offset) * Scale);
    
        return pointOnModel;
    }
    ```
    
3. **Retrieve the view range offsets** – If the viewport’s view is a plan view, we extract the top and bottom clip plane offsets.
    
    ```csharp
    var view = viewPort.ViewId.GetElement<View>() as ViewPlan;
    var range = view.GetViewRange();
    var maxZ = range.GetOffset(PlanViewPlane.TopClipPlane);
    var minZ = range.GetOffset(PlanViewPlane.ViewDepthPlane);
    ```
    
4. **Construct a 3D bounding outline** – Using the projected points and the view range offsets, we define a 3D outline. This step ensures that elements spanning different Z-levels are considered.  
      
    Additionally, a **tolerance buffer** is applied to the `minZ` and `maxZ` values. This ensures that elements precisely on the bounding box edges—such as **floors with thickness extending below the minimum point**—are still included in the selection. Without this, elements that slightly exceed the defined limits would be excluded.
    
    ```csharp
    double tolerance = 2;
    p0 = new XYZ(p0.X, p0.Y, minZ - tolerance);
    p1 = new XYZ(p1.X, p1.Y, maxZ + tolerance);
    ```
    
    This adjustment guarantees that **structural elements like floors, slabs, and other components touching the boundary remain part of the selection.**
    
5. **Apply a bounding box filter** – Now that we have the 3D outline, we use it to create a `BoundingBoxIsInsideFilter`, which filters elements inside the defined bounding box.
    
    ```csharp
    var bbxFilter = new BoundingBoxIsInsideFilter(outline);
    var elements = new FilteredElementCollector(Doc, view.Id)
        .WherePasses(bbxFilter)
        .ToElements();
    ```
    
6. **Select the retrieved elements** – Finally, we update the UI selection to highlight the filtered elements.
    
    ```csharp
    UiDoc.Selection.SetElementIds(elements.Select(o => o.Id).ToList());
    ```
    

Final Complete code block

```csharp
var viewPort = UiDoc.PickElement<Viewport>();
var outline = viewPort.GetBoxOutline();
var p0 = outline.MinimumPoint.ConvertToModelSpace(viewPort);
var p1 = outline.MaximumPoint.ConvertToModelSpace(viewPort);
var view = viewPort.ViewId.GetElement<View>() as ViewPlan;

var range = view.GetViewRange();
var maxZ = range.GetOffset(PlanViewPlane.TopClipPlane);
var minZ = range.GetOffset(PlanViewPlane.ViewDepthPlane);
double tolerance = 2;
p0 = new XYZ(p0.X, p0.Y, minZ - tolerance);
p1 = new XYZ(p1.X, p1.Y, maxZ + tolerance);

outline = new Outline(p0, p1);
var bbxFilter = new BoundingBoxIsInsideFilter(outline);
var elements = new FilteredElementCollector(Doc, view.Id)
    .WherePasses(bbxFilter)
    .WhereElementIsNotElementType()
    .Where(x => x.Category != null && x.Category.HasMaterialQuantities)
    .Where(x => x.Category != null && x.Category.CategoryType != CategoryType.Internal)
    .Where(x => x.Category != null && x.Category.CategoryType != CategoryType.AnalyticalModel);
List<ElementId> ids = elements.Select(o => o.Id).ToList();
UiDoc.Selection.SetElementIds(ids);
```

This implementation should help you gather the elements that are actually visible within the viewport.