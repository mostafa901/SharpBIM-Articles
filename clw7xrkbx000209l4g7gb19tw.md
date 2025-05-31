---
title: "Offset Ellipse"
datePublished: Wed May 15 2024 14:47:37 GMT+0000 (Coordinated Universal Time)
cuid: clw7xrkbx000209l4g7gb19tw
slug: offset-ellipse
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1715784700831/265359ed-c896-414e-a808-9362e150d777.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1715784715788/04ec6095-40a5-4257-b3db-46bcda851cf8.png
tags: converter, offset, revitapi, hermite-spline, ellipse

---

While using the offset function in the Revit API, I noticed that [ellipses](https://www.revitapidocs.com/2022/b966b82f-0627-c94a-9f37-994d00bdff18.htm) don't retain their properties after offsetting. Instead, they are converted to another type of segment classified as a [Hermite-spline](https://www.revitapidocs.com/2022/6852ca4c-2fad-cda1-be75-54e712a39318.htm). How can I keep the ellipse properties after offsetting?

From my experience, the best approach is to recreate the ellipse with the offset value. If the curve is unbound, meaning it is a fully closed ellipse, we can create another ellipse with an offset of the original. Simply take the offset value and add it to the original X and Y radius.

```csharp
double offset = 1;
var newEllipse = Ellipse
    .CreateCurve(
        originalEllipse.Center,
        originalEllipse.RadiusX + offset,
        originalEllipse.RadiusY + offset,
        originalEllipse.XDirection,
        originalEllipse.YDirection,
        0,
        10
    );
```

For a bound ellipse (a partially drawn ellipse), we need to get the start and end parameters to use them in the new creation method.

```csharp
double offset = 1;
var newEllipse = Ellipse
      .CreateCurve(
          originalEllipse.Center,
          originalEllipse.RadiusX + offset,
          originalEllipse.RadiusY + offset,
          originalEllipse.XDirection,
          originalEllipse.YDirection,
          originalEllipse.GetEndParameter(0),
          originalEllipse.GetEndParameter(1)
      );
```

This method works well when you are only dealing with an ellipse. But what if you need to offset a series of connected curves that include different types of curves?

Since an ellipse is a conic section and can be represented by arcs, we can extract arcs from the ellipse by dividing it into segments and representing each segment as an arc. The number of segments you choose will affect the accuracy of the result. More segments will give a smoother representation but require more computational resources. We can also optimize the result afterward. Let's call this approach A.

Another approach, "B," is to use the [Tesselate function](https://www.revitapidocs.com/2022/f95f3199-3251-c708-c5a3-a0e9ef95ecfa.htm). Tesselate will return points that approximate the curve, but only if the curve is bound. From these points, we can start drawing arcs. Although this method will give the most accurate result, it will consume more computational resources.

I have conducted some analysis and performance tests on two curves: one is an ellipse, and the other is a Hermite spline, both with a length of 10920.9 mm. I found that converting any curve to arcs using the LOD "Approach A" provides better-optimized output and less performance overhead compared to the other approach. See the results below:

| Conversion Method | Source | Time elapsed (milliseconds) | Number of Arcs |
| --- | --- | --- | --- |
| LOD : 10 segments | Ellipse | 16 | 10 |
|  | Hermite Spline | 46 | 37 |
| Tesselate : 49 points | Ellipse | 16 | 10 |
|  | Hermite Spline | 49 | 37 |

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715781705948/f59c0d58-13b7-4fcf-a4f8-01ce66cac7ac.png align="center")

The question that might raise, why I would need to convert Hermite spline to arcs. one of the reasons is you can not place walls over Hermite-Spline.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715781958842/b19e4ec6-adaa-4527-a1cf-f4606a150f84.gif align="center")

I have place the code, in case you are keen to know how i come up to all of the above:

```csharp
 // Convert Curve to arc Tesselate
        var stopWatch = Stopwatch.StartNew();
        List<Arc> arcs = new();

        var points = originalEllipse.Tessellate();

        double count = points.Count;
        for (int i = 2; i < count; i += 2)
        {
            var start = points[i - 2];
            var middle = points[i - 1];
            var end = points[i];
        
            var arc = Arc.Create(start, end, middle);
            if (arcs.Any())
            {
                var lastArc = arcs.Last();
                if (lastArc.Center.IsAlmostEqualTo(arc.Center))
                {
                    arcs[arcs.Count - 1] = Arc.Create(
                        lastArc.GetEndPoint(0),
                        end,
                        lastArc.GetEndPoint(1)
                    );
                }
                else
                {
                    arcs.Add(arc);
                }
            }
            else
                arcs.Add(arc);
        }
        stopWatch.Stop();
        
        var arcLoop = CurveLoop.Create(arcs.Cast<Curve>().ToList());
        Doc.Create.NewDetailCurveArray(UiDoc.ActiveGraphicalView, arcLoop.ToCurveArray());
```

```csharp
 // Convert Curve to arc LOD"
        var stopWatch = Stopwatch.StartNew();
        List<Arc> arcs = new();

        double lod = 10; // level  of detail
        
        double count = lod * 2;
        for (int i = 2; i <= count; i += 2)
        {
            var start = originalEllipse.Evaluate((i - 2) / count, true);
            var middle = originalEllipse.Evaluate((i - 1) / count, true);
            var end = originalEllipse.Evaluate(i / count, true);

            var arc = Arc.Create(start, end, middle);
            if (arcs.Any())
            {
                var lastArc = arcs.Last();
                if (lastArc.Center.IsAlmostEqualTo(arc.Center))
                {
                    arcs[arcs.Count - 1] = Arc.Create(
                        lastArc.GetEndPoint(0),
                        end,
                        lastArc.GetEndPoint(1)
                    );
                }
                else
                {
                    arcs.Add(arc);
                }
            }
            else
                arcs.Add(arc);
        }
        stopWatch.Stop();
        
        var arcLoop = CurveLoop.Create(arcs.Cast<Curve>().ToList());
        Doc.Create.NewDetailCurveArray(UiDoc.ActiveGraphicalView, arcLoop.ToCurveArray());
```