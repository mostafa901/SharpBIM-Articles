---
title: "Face of a linked element"
datePublished: Tue Jun 04 2024 22:00:00 GMT+0000 (Coordinated Universal Time)
cuid: clx5t60xh000108mh8pyxfal0
slug: face-of-a-linked-element
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1717832279259/d832d897-a021-40d1-af9c-5576df01e06b.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1717832286030/e30ccc8e-0e53-4c2e-ad90-a4673e63bca9.png
tags: face, revit-api, face-geometry, linked, linkedelement

---

This article is a continuation of my previous article, "[Highlight Linked Element](https://hashnode.com/post/clw3qi0zu00000akybbdt6fc0)." Here, we will explore how to place a face-based family on a face of an element within a linked document in Revit.

The concept builds on the ideas from my previous article but includes additional implementations. When we place a family on a face, the relationship is established via a Reference. This means that the created family instance will register the host element by its reference. [Scott Wilson provides an excellent explanation](https://thebuildingcoder.typepad.com/blog/2016/04/stable-reference-string-magic-voodoo.html) (blogged by Jeremy Tammik) that clarified my understanding of how a reference is structured.

With that foundation, let’s dive into the code that illustrates how to obtain a reference to a face of an element in a linked document.

### Step-by-Step Process

1. **Pick an Element by Point:** Use `UiDocument.Selection.PickElement(ObjectType.PointOnElement)` to select an element by a point.
    
2. **Extract the Necessary Information:** After selection, you will have:
    
    * The a translated reference of the face in the host document
        
    * The Revit link instance
        
    * The Revit link document
        
    * The original element selected (not the face itself)
        
3. **Loop Through Geometry Faces:** With the gathered information, loop through the geometry faces and compare each face's reference to the reference you selected.
    
4. **Handle Translated References:** Note that the reference you picked is translated to align with the host document and the linked Revit instance. Use the following statement to restore the reference to its original form `originalReference = linkedReference.CreateReferenceInLink();`
    

### Code Example

Below is an example code snippet that highlights the face as a test, ensuring the Reference we picked is the expected face.

```csharp
var linkedReference = UiDoc.Selection.PickObject(ObjectType.PointOnElement);
var rvtInstance = linkedReference.ElementId.GetElement<RevitLinkInstance>();
var originalReference = linkedReference.CreateReferenceInLink();
var linkedDocument = rvtInstance.GetLinkDocument();
var originalElement = linkedDocument.GetElement(linkedReference.LinkedElementId);

var geo = originalElement.get_Geometry(
    new Options()
    {
        DetailLevel = ViewDetailLevel.Undefined,
        IncludeNonVisibleObjects = true,
        ComputeReferences = true
    }
);

Face seekedFace = null;
var solids = geo.OfType<Solid>().ToList();
// incase the selected element is solid then we need to get the solids
// from the GeometryInstances
solids.AddRange(geo.OfType<GeometryInstance>().SelectMany(o=> o.SymbolGeometry.OfType<Solid>()));

string originalReferenceString = originalReference.ConvertToStableRepresentation(
    linkedDocument
);
foreach (var solid in solids)
{
    if (seekedFace != null)
        break;
    foreach (Face face in solid.Faces)
    {
        if (face.Reference == null)
            continue;
        if (
           originalReferenceString.EndsWith( face.Reference.ConvertToStableRepresentation(linkedDocument))
        )
        {
            seekedFace = face;
            break;
        }
    }
}
if (seekedFace != null)
{
    // now we have the face, we can do what we need with that
    // I reTranslated the reference back to the host document
    // to cross check the above works as it should
    var highlightReference = seekedFace.Reference.CreateLinkReference(rvtInstance);
    UiDoc.Selection.SetReferences([highlightReference]);
}
else
{
    // Face not Found :(
}    
```

One effective use of selecting a face of a linked model is placing a face-based hosted family on that linked face. Since the reference is already established, you can utilize this reference [as an argument when creating a face-based family](https://www.revitapidocs.com/2016/be4b822c-829a-7e7b-8c03-a3a324bfb75b.htm). For instance, by picking a face on a linked element, you can directly host a lighting fixture or other face-based families, ensuring precise alignment with the linked model. Below is an example that illustrates this process.

```csharp
var famSymbol = // you need to know what familySymbole
 
Trans.Start(); // start transaction
Doc.Create.NewFamilyInstance(
    linkedFaceReference,
    linkedFaceReference.GlobalPoint,
    XYZ.BasisZ,
    famSymbol
);
Trans.Commit(); // commit Transaction
```