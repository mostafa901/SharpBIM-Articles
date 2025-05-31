---
title: "ModelText Rotation"
datePublished: Mon May 06 2024 05:20:35 GMT+0000 (Coordinated Universal Time)
cuid: clvuijopb000f08l4cp2x0rtn
slug: modeltext-rotation
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1715612682961/0510e74d-7712-424b-8ce5-5b997676dac9.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1715612693352/c5dfca0a-bfdf-4cf1-a961-ef39886c9d9b.png
tags: direction, rotation, revitapi, model-text, angle

---

Model text, in fact, is a bit tricky to determine its rotation value. I attempted to inspect its solids in the hope of finding any reference lines to grasp it, but I was unsuccessful. As I expanded my investigation to comprehend how model text functions, I discovered some hidden references. These references are visible on the user interface only when you attempt to align the model text or draw a line over it.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1714971432078/b651602d-3b07-46ad-885d-157162dbaf1a.png align="center")

Which I couldn’t figure out how to identify these references. But wait, here comes the tricky part. When I attempted to obtain the geometry of this model text element, I got 2 solids. The first one has faces and edges, and the second one has zero faces and edges. I thought the second empty solid was indeed empty, but it turned out not to be. When I applied SolidUtil.SplitVolume to the empty solid, it produced 8 solids with actual volumes \[might be a bug, @Jermy Tamik to confirm\].

Continuing with this account, the 8 returned solids are single-faced solids. These faces represent the references that I managed to select while hovering the mouse over. Through experimentation, I realized that the first solid returned always indicates the direction of the text.

Now the rest is just a couple of line code to get the angle. however, it worth mentioning, such procedure needs to be tested that would return the expected results in all cases. I couldn't find a supporting documentation for such workaround, but it WORKS :)

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1714971444590/814e29a6-89d7-4f5d-88da-b6a80134b28c.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1714972339856/02d73e92-c5ba-4c6b-a33f-9b617a0d02b9.png align="center")

```csharp
public IEnumerable<Solid> GetSolids(ModelText modelTextElement)
{
    var op = new Options()
    {
        IncludeNonVisibleObjects = true
    };

    var geomCollection = modelTextElement.get_Geometry(op);

    IEnumerable<Solid> solids = geomCollection.OfType<Solid>();
    return solids;
}

public double? GetModelTextDirection(ModelText modelTextObject)
{
    double? angleInDegree = null;
    if (modelTextObject != null)
    {
        var solids = GetSolids(modelTextObject);
        foreach (var solid in solids)
        {
            // we are keen to get the reference solids, hence ignore all above zero volume value
            if (solid.Volume > 0)
                continue;

            var directedSolid = SolidUtils.SplitVolumes(solid).First();
            var faces = directedSolid.Faces.Cast<Face>();

            PlanarFace directedFace = faces.First() as PlanarFace;
            var direction = directedFace.Normal();

            double toDegree = 180 / Math.PI;
            angleInDegree = Math.Round(XYZ.BasisX.AngleTo(direction) * toDegree, 2);
            break;
        }

    } 
    return angleInDegree;
}
```