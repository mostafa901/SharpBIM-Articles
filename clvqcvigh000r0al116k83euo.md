---
title: "Purging Viewport Types"
datePublished: Fri May 03 2024 07:30:44 GMT+0000 (Coordinated Universal Time)
cuid: clvqcvigh000r0al116k83euo
slug: purging-viewport-types
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1715614153970/05e951cd-91aa-4fdf-9679-b02f89122a05.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1715614164133/2a57e8ed-b39b-40d9-af4d-b329da73ca87.png
tags: revitapi, purge, viewports

---

In RevitAPI, filtering and collecting elements are present in almost every developed code for writing add-ins. Usually, we locate elements by their `Category` or `class` type using statements like this, for example when we want to retrieve all walls from the current document.

```csharp
var elements = new FilteredElementCollector(doc).OfClass(typeof(Wall));
var elements = new FilteredElementCollector(doc).OfCategory(BuiltInCategory.OST_Walls);
```

The process becomes trickier when trying to find an element without a defined class, like Viewport Types. When we inspect a viewport using Revitlookup, we discover that the ViewportType is categorized as ElementType.

ElementType serves as a fundamental class for all Revit elements; hence, it is not helpful. Considering alternative approaches, one could explore the possibility of identifying purgable viewports indirectly by targeting existing parameters. For example, since ViewportType is a system family all must possess the "Show Extension Line" parameter `BuiltInParameter.VIEWPORT_ATTR_SHOW_EXTENSION_LINE` , we can use this parameter as a filter rule to identify all types that possess it. Fortunately, this parameter is a bool parameter, whose value is either 0 or 1, then we can use [`ParameterFilterRuleFactory`](https://www.revitapidocs.com/2016/f17be87a-a682-5fcc-8ebc-1449952461ff.htm)`.`[`CreateGreaterOrEqualRule`](https://www.revitapidocs.com/2016/f17be87a-a682-5fcc-8ebc-1449952461ff.htm) and supply a value of 0 for checking.

With a list of viewport types established, the next step involves iterating through each type to retrieve the associated viewports using the [**GetDependentElements(ElementFilter)** method.](https://www.revitapidocs.com/2024/56e875d3-014b-a996-69c3-e6ed9b885f5c.htm)

Here's an example to illustrate the process:

```csharp
// https://forums.autodesk.com/t5/revit-api-forum/deleting-unpurgeable-viewport-types-through-api/td-p/8790238
[Description("Purge Viewports")]
public void PurgeViewPorts()
{
    try
    {
        var doc = RvtApi.Doc;
        // create filterRule to be used in collection
        var rule = ParameterFilterRuleFactory.CreateGreaterOrEqualRule(
            new ElementId(BuiltInParameter.VIEWPORT_ATTR_SHOW_EXTENSION_LINE),
            0
        );

        // get all elements that comply to this rule
        var viewPortTypes = new FilteredElementCollector(doc)
            .WhereElementIsElementType()
            .OfCategory(BuiltInCategory.INVALID)
            .WherePasses(new ElementParameterFilter(rule));

        // now for each viewportType find a viewport instance dependent
        List<ElementId> purgeableIds = new List<ElementId>();
        foreach (var viewportType in viewPortTypes)
        {
            var viewports = viewportType.GetDependentElements(
                new ElementCategoryFilter(BuiltInCategory.OST_Viewports)
            );
            if (!viewports.Any())
            {
                purgeableIds.Add(viewportType.Id);
            }
        }

        // show a list of all purgeable Ids
        MessageBox.Show(
            string.Join("\r\n", purgeableIds.Select(o => o.Value().ToString()))
        );
        // ... handle these ids
    }
    catch (Exception ex)
    {
        // something bad happened
    }

    return;
}   
```

Update: 30.12.24

I recently [read of another cleaner](https://forums.autodesk.com/t5/revit-api-forum/change-viewport-type-via-api/m-p/13234218/highlight/true#M83138) way to fetch these viewport Types, I think i would be more inclined to the below code, less code, simpler, and easier to maintain.

```csharp
FilteredElementCollector collector = new FilteredElementCollector(doc)
    .OfClass(typeof(FamilySymbol))
    .OfCategory(BuiltInCategory.OST_ViewportLabel);
```