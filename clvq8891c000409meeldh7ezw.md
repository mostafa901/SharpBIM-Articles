---
title: "Getting new Elements"
seoTitle: "Get new elements without Document Changed"
seoDescription: "another approach to catch newly created elements without attaching to document changed"
datePublished: Fri May 03 2024 05:20:40 GMT+0000 (Coordinated Universal Time)
cuid: clvq8891c000409meeldh7ezw
slug: getting-new-elements
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1715614275266/d7f41521-9386-4bd9-8aa8-0aca8e3141d1.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1715614282391/4745c156-c033-41c1-93d7-805a7025e688.png
tags: revitapi, elementid, document-changed

---

Each time we create or draw an element in Revit, a new record is added to the Revit database. This could be a wall, a window, a floor, and so on. These new records are identified by an ID, which is a number stored in a class called [ElementId](https://www.revitapidocs.com/2020/d9848d7d-5917-2433-8454-f65f5ac03964.htm).

When we develop an add-in that generates elements, these elements receive new IDs. Creating elements through code can take two forms: either using Document.Create or through Prompts (Prompts is not the same as [PostableCommand](https://www.revitapidocs.com/2020/f6ccdc1b-6ac3-9c49-d0bb-8a7d1877eab0.htm)). For instance, creating a dimension using `Document.NewDimension(View, Line, ReferenceArray)` through Document.Create results in a Dimension instance.

Obtaining the new ID from [Document.Create](https://www.revitapidocs.com/2020/4f835512-a922-c7da-d389-3bdcb41a5660.htm?section=methodTableToggle) is straightforward because the method returns the newly created element. However, dealing with Prompts is a bit more complex.

Prompts involve handing over control to the user to place and create elements before returning to the code session. This disconnection of sessions restricts access to the created elements. To work around this, one approach is to gather all the newly created IDs from the project, re-collect them after the session ends, and then filter out the existing IDs to identify the newly created elements. Below is a snippet I have created for this scenario:

```csharp
FamilyInstanceFilter familyInsFilter = new FamilyInstanceFilter(symbol.Document, symbol.Id);
// collect all existing instanced and store their ids
var oldIds = new FilteredElementCollector(symbol.Document, symbol.Document.ActiveView.Id)
	.WherePasses(familyInsFilter)
	.ToElementIds();
try
{
// Ask user to place a familyInstance
	uiDoc.PromptForFamilyInstancePlacement(symbol);
}
catch (Autodesk.Revit.Exceptions.OperationCanceledException ex)
{
	// user has exited the operation
}
catch (Exception ex)
{
	// something bad happened abort
	return;
}
// Again collect all existing instances, and compare it to the old collection
var newFamilyInstanceIds = new FilteredElementCollector(symbol.Document, symbol.Document.ActiveView.Id)
	.WherePasses(familyInsFilter)
	.ToElementIds()
	.Except(oldIds);  
```

In the above example, I used a [prompt to create familyinstance](https://www.revitapidocs.com/2020/619d8d3f-ac64-26bf-cd82-0f6c37221367.htm) that essentially asks the user to choose where to position these familyInstances. After creating them, we compare the collection results to extract the newly created family IDs.