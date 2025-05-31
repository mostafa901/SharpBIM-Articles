---
title: "Highlight elements from a linked document"
datePublished: Fri May 10 2024 22:00:00 GMT+0000 (Coordinated Universal Time)
cuid: clw3qi0zu00000akybbdt6fc0
slug: highlight-elements-from-a-linked-document
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1715612758494/715f8a3a-7c0e-4696-bbd4-f07d5d83bbbe.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1715612774071/a3b9f46f-d33d-4270-9c50-d177aa0f2ec4.png
tags: revitapi, highlight-element, linked-element, highlight-linked-element

---

It has long been a wish to select elements from a linked element. This wish seems to have been granted in the Revit 2023 Release API.

A new Selection function called [`SetReferences`](https://www.revitapidocs.com/2023/813a9d31-bc4f-1ebc-9a7b-69a2a99d22ac.htm) has been added, allowing elements to be highlighted via reference. We don't often use references to highlight elements, but rather to set hosts, like placing hosted families or extracting element IDs from a [`ReferenceIntersector`](https://www.revitapidocs.com/2023/36f82b40-1065-2305-e260-18fc618e756f.htm) or when selecting by picking.

So, if we provide the `SetReferences` function with references from a linked document, will it work? In theory, yes, it should work. However, some extra work is required to capture such element references. Firstly, we need to understand that this function operates on the currently active document. This means that the references we provide must be in a format that the current document can recognize to highlight them in the current view.

Let's attempt to highlight a face from an element in a linked document in the following steps:

1. Click on a point over one of the faces in a linked document.
    
2. Then, pass this reference to `SetReferences`, and it will highlight the face from the linked document.
    
3. Similarly, if you press Tab to cycle through line, face, and object, once you reach the object, select it to get the object reference.
    

```csharp
var linkedFaceReference = UiDoc.Selection.PickObject(
    Autodesk.Revit.UI.Selection.ObjectType.PointOnElement
);
UiDoc.Selection.SetReferences([linkedFaceReference]);
```

Now, this only happens when a user interacts with the UI. But what if I have an element ID from a linked document that I want to highlight? The real question then becomes, how can I extract a reference from an `ElementId` that belongs to a linked document?

This is achievable, but not directly from the `ElementId`; we need to work with the element itself. First, we need to get the element from the linked document, then create a reference for this element. However, we can't use this reference as it's only meaningful to the linked document, not the current one. Therefore, we must convert this reference to the current document using `CreateLinkReference` and `RevitLinkInstance`. It might sound confusing, but I've included the code below to demonstrate how it functions clearly.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715541666190/cb18359c-2a8d-4ddf-bcae-ec0a08be1911.gif align="center")

```csharp
var pickedReference = UiDoc.Selection.PickObject(
    Autodesk.Revit.UI.Selection.ObjectType.PointOnElement
);

// get Revit link Instance and its document
var linkedRvtInstance = Doc.GetElement(pickedReference) as RevitLinkInstance;
var linkedDoc = linkedRvtInstance.GetLinkDocument();

//get the Linked element from the linked document
var linkedElement = linkedDoc.GetElement(pickedReference.LinkedElementId);

// now create a reference from this element [ this is a reference inside the linked document]
var reference = new Reference(linkedElement);

// convert the reference to be readable from the current document
reference = reference.CreateLinkReference(linkedRvtInstance);

// now the linked element is highlighted
UiDoc.Selection.SetReferences([reference]);
```