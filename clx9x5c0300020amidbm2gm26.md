---
title: "Render Text via Direct Context 3D"
datePublished: Tue Jun 11 2024 04:45:34 GMT+0000 (Coordinated Universal Time)
cuid: clx9x5c0300020amidbm2gm26
slug: render-text-via-direct-context-3d
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1718081100646/6d13730c-2163-4530-9273-a8107dff25a9.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1718081108030/b86883c4-e9ea-445a-8f8b-e3216d0be00d.png
tags: 3d, revitapi, render-text, direct-context-3d

---

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718134364135/81e2206d-ef6b-4d54-a901-578c2e94027b.png align="center")

Rendering text in the Revit API using [`DirectContext3D`](https://www.autodesk.com/autodesk-university/class/DirectContext3D-API-Displaying-External-Graphics-Revit-2017) can be approached in several ways, each with its advantages and limitations. Here, we explore two methods: [`FormattedText`and `GraphicsPath`](https://learn.microsoft.com/en-us/dotnet/desktop/wpf/advanced/drawing-formatted-text?view=netframeworkdesktop-4.8).

#### [FormattedText](https://learn.microsoft.com/en-us/dotnet/desktop/wpf/advanced/drawing-formatted-text?view=netframeworkdesktop-4.8)

`FormattedText.BuildGeometry` is a practical choice for rendering text. It converts text into a geometry that can be rendered in 3D. However, it doesn't always match the font shape perfectly, but it offers acceptable performance for many applications.

**Advantages:**

* Better performance.
    
* Simpler implementation.
    
* Generates a reasonable approximation of text shapes.
    

**Disadvantages:**

* Limited accuracy in matching the font's exact shape.
    
* Converts curves to polyline segments, which might not always be desirable.
    

**Implementation Example:**

```csharp
List<CurveLoop> rvtCurves = [];

    var formatted = fontSettings.GetFormattedText(message);
    GeometryGroup geo = formatted.BuildGeometry(new System.Windows.Point()) as GeometryGroup;
    var pathGeometry = geo.GetFlattenedPathGeometry();
 
    foreach (var fig in pathGeometry.Figures)
    {
        var cloop = new CurveLoop();

        XYZ startPoint = fig.StartPoint.ToRvtPoint();
        for (int i = 0; i < fig.Segments.Count; i++)
        {
            PathSegment seg = fig.Segments[i];
            try
            {
                Curve curve = null;
                if (seg is LineSegment lineSeg)
                {
                    curve = Line.CreateBound(startPoint, lineSeg.Point.ToRvtPoint());
                }
                else if (seg is BezierSegment bezSeg)
                {
                    var p0 = startPoint;
                    var p1 = bezSeg.Point1.ToRvtPoint();
                    var p2 = bezSeg.Point2.ToRvtPoint();
                    var p3 = bezSeg.Point3.ToRvtPoint();
                    curve = HermiteSpline.Create([p0, p1, p2, p3], false);
                }
                else if (seg is PolyLineSegment polSeg)
                {
                    var points = polSeg.Points.Select(o => o.ToRvtPoint()).ToList();
                    points.Insert(0, startPoint);
                    foreach (var item in points.DrawAPI(false))
                    {
                        cloop.Append(item);
                    }
                }
                else if (seg is PolyBezierSegment polBezSeg)
                {
                    var points = polBezSeg.Points.Select(o => o.ToRvtPoint()).ToList();
                    points.Insert(0, startPoint);
                    var tangents = new HermiteSplineTangents();
                    tangents.StartTangent = (points[1] - points[0]).Normalize();
                    tangents.EndTangent = (points[3] - points[2]).Normalize();
                    curve = HermiteSpline.Create([points[0], points[3]], periodic: false, tangents);
                    cloop.Append(curve);
                }
                else
                {
                    // Unsupported type yet
                }

                if (curve != null)
                {
                    cloop.Append(curve);
                    startPoint = curve.GetEndPoint(1);
                }
            }
            catch (Exception ex)
            {
                // something went wrong
            }
        }
        
        rvtCurves.Add(cloop);
    }
}
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718079488199/471b9185-f72e-42e1-9c9d-1419166dd6bb.png align="center")

[GraphicsPath](https://learn.microsoft.com/en-us/dotnet/api/system.drawing.drawing2d.graphicspath?view=net-8.0)

`GraphicsPath` offers a closer match to the original font shapes but at a significant cost in terms of performance. It generates a higher number of points and curves, leading to a denser and more accurate geometry representation.

Red is FormattedText  
Blue is GraphicsPath

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718079561748/50375b22-3876-4788-88a7-c8bc060880dd.png align="center")

**Advantages:**

* Higher accuracy in font shape rendering.
    
* Better for applications where visual fidelity is critical.
    

**Disadvantages:**

* Higher computational cost.
    
* Increased number of faces and points can lead to performance issues.
    

Implementation example

```csharp
List<CurveLoop> rvtCurveLoops = [];
try
{
    // Create a GraphicsPath object
    GraphicsPath path = new GraphicsPath();
    path.AddString(
        message,
        new System.Drawing.FontFamily(fontSettings.Family.FamilyNames.First().Value),
        (int)fontSettings.FontStyle,
        fontSettings.FontSize,
        new PointF(),
        StringFormat.GenericDefault
    );

    // Create lists to hold points and types
    PointF[] points = path.PathPoints;
    byte[] types = path.PathTypes;
   
    // Initialize a list to hold the resulting Revit curves
    List<Curve> revitCurves = new List<Curve>();

    // Temporary lists to store points for each curve
    List<XYZ> linePoints = new List<XYZ>();
    List<XYZ> BezierPoints = new List<XYZ>();

    XYZ startPoint = points[0].ToRvtPoint();
  
    for (int i = 1; i < points.Length; i++)
    {
        // Convert PointF to XYZ
        XYZ point = points[i].ToRvtPoint();
        byte type = types[i];
      
        switch (type)
        {
            case (byte)PathPointType.Start:

                startPoint = point;
                if (revitCurves.Any())
                {
                    rvtCurveLoops.Add( CurveLoop.Create(revitCurves));
                    revitCurves.Clear();
                }
                break;

            case (byte)PathPointType.Line:
                linePoints.Add(startPoint);
                linePoints.Add(point);
                startPoint = linePoints.Last();

                AddLineCurves(linePoints, revitCurves);
                linePoints.Clear();
                break;

            case (byte)PathPointType.Bezier3:
                BezierPoints.Add(startPoint);
                BezierPoints.Add(point);
                BezierPoints.Add(points[++i].ToRvtPoint());
                BezierPoints.Add(points[++i].ToRvtPoint());
                startPoint = BezierPoints.Last();
                AddBezierCurves(BezierPoints, revitCurves);
                BezierPoints.Clear();
                break;

            case (byte)PathPointType.CloseSubpath:
                   // $"{PathPointType.CloseSubpath} not implemented"
                break;

            case (byte)PathPointType.PathMarker:
                    //$"{PathPointType.PathMarker} not implemented"
                break;

            case (byte)PathPointType.DashMode:
                    //$"{PathPointType.DashMode} not implemented"
                break;

            default:
                // Handle any other types if necessary
                break;
        }
    }
    if (revitCurves.Any())
    {
         rvtCurveLoops.Add( CurveLoop.Create(revitCurves));
    }
 
    return rvtCurveLoops;
```

**Performance Comparison:**

* **GraphicsPath Method:**
    
    * Points: 104
        
    * Curves: 35
        
* **FormattedText Method:**
    
    * Points: 47
        
    * Curves: 47
        

**Example Results:**

* **Red:** FormattedText output.
    
* **Blue:** GraphicsPath output.
    
* font used: [Ink Free](https://www.dafontfree.co/ink-free-font/)
    

In conclusion, while `FormattedText.BuildGeometry` offers better performance, `GraphicsPath` provides more accurate text shapes. The choice between them depends on the specific requirements of accuracy versus performance in your application.

Red is FormattedText  
Blue is GraphicsPath

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718079554733/f6682918-80f0-4149-a554-1a2239072156.png align="center")

However using GraphicsPath, usually returns a broken graphics, not sure why yet.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718081491849/5dfc991a-524b-480b-8983-83b475a256f3.png align="center")

Edit 11th June

I was able to realize the why FormattedText shows different optimized shape than GraphicsPath, due to the face we are using this statement

```csharp
var pathGeometry = geo.GetFlattenedPathGeometry();
```

This statement [accoriding to Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.windows.media.geometry.getflattenedpathgeometry?view=windowsdesktop-8.0) The polygonal approximation of the geometry. thus if we changed the tollerance of flattening, it would return a better result.

```csharp
 var figures = geo.GetFlattenedPathGeometry(.00001, ToleranceType.Relative)
                  .Figures;
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718121398377/cd550072-c554-4252-8009-88a565e7b892.png align="center")

This is using FormattedText with flattengeometry of .00001 almost the maximum. which returned exactly what is expected

Now we play with tolerance to suit the performance against quality as it is always every developer challenge

Forgot to mention... that when converting to solid, then we can have what the font size is actually rendered on revit. due to the fact that each font scales differently, we can use [Transform.ScaleBasis(value](https://www.revitapidocs.com/2017/35360886-77c5-4117-e395-b83b95f9c884.htm)), in the direct context 3D, to scale it to the size we need saying `intendedWidth` should be 1 meter long

```csharp
// extention function are helpful here
scale = intendedWidth/textSolid.GetBoundingBox().Width;
```

an example of how text can be helpful during design

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718134373433/6fce6b08-a49a-4c97-b70b-3c11000be9eb.png align="center")