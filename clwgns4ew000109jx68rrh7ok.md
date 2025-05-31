---
title: "What Command is Executed?"
datePublished: Tue May 21 2024 17:18:02 GMT+0000 (Coordinated Universal Time)
cuid: clwgns4ew000109jx68rrh7ok
slug: what-command-is-executed
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1716311776131/fd5e5635-e124-47cf-8b34-975ac88ca976.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1716311782706/1a3a7be3-1cc3-4d84-a30b-1b59a6525db9.png
tags: revitapi, commandid, postablecommandid

---

In Revit development, it can be challenging to determine which specific [PostableCommandId](https://www.revitapidocs.com/2017/f6ccdc1b-6ac3-9c49-d0bb-8a7d1877eab0.htm) has been executed. While we can identify which button has been triggered and extract certain constants, correlating these to a specific [PostableCommandId](https://www.revitapidocs.com/2017/f6ccdc1b-6ac3-9c49-d0bb-8a7d1877eab0.htm) remains elusive. [The documentation provides](https://help.autodesk.com/view/RVT/2025/ENU/?guid=Revit_API_Revit_API_Developers_Guide_Advanced_Topics_Commands_html) a description of PostableCommandId, yet it does not offer a direct method for this correlation.

One potential solution is to create a mapping system by extracting the executed button's Id and binding it to a PostableCommandId. This article explores a method to achieve this by monitoring button executions and building a dictionary to track these commands. By using Autodesk's ComponentManager and RevitCommandId classes, developers can effectively capture and map executed commands, simplifying the monitoring process. Here, I will demonstrate how to extract button IDs and create a command map to efficiently track button executions in Revit.

I believe there are many benefits to tracking which button is executed or which one we need to track. For example, to maintain a clean Revit model, we can allow drafters to sync their model only after certain rules have been met. Even better, we can trigger the execution of specific commands before saving a project.

```csharp
// run this from any method to monitor buttons executions one time only is needed
Autodesk.Windows.ComponentManager.ItemExecuted += ComponentManager_ItemExecuted;

private Dictionary<string, PostableCommand> IdCommandMap = [];

private void BuildCommandMap()
{
    var commands = Enum.GetValues(typeof(PostableCommand)).Cast<PostableCommand>();
    foreach (var command in commands)
    {
        IdCommandMap.Add(
            RevitCommandId
                .LookupPostableCommandId(command)
                .Name,
            command
        );
    }
}

private void ComponentManager_ItemExecuted(
    object sender,
    Autodesk.Internal.Windows.RibbonItemExecutedEventArgs e
)
{
    if (IdCommandMap.TryGetValue(e.Item.Id, out PostableCommand executedCommand))
    {
        // the Catched command
    }
    else
    {
        // command is from an addin " not a built in Revit command"
    }
}
```

One more last thing, if the intention to get buttons details then we can extract using the below example, and using the [postcommand](https://www.revitapidocs.com/2015/b0df464d-1733-ea9e-ac40-399fa9c9a037.htm) we can use the Id property to invoke the button execution.

```csharp
var tabs = Autodesk.Windows.ComponentManager.Ribbon.Tabs;
StringBuilder sb = new StringBuilder();

for (int i = 0; i < tabs.Count; i++)
{
    var tab = tabs[i];
    foreach (var panel in tab.Panels)
    {
        foreach (var button in panel.Source.Items)
        {
            sb.AppendLine($"Name:          {button.Text}");
            sb.AppendLine($"id:            {button.Id}");
            sb.AppendLine($"hotkey:        {button.KeyTip}");
            sb.AppendLine($"image:         {button.LargeImage}");
            sb.AppendLine($"large Image:   {button.Image}");
        }
    }
}
```

Another aspect to consider is how to get the command that was executed before running the IExternalCommand object.

You can do this by registering for the `ComponentManager.PreviewExecute` event anywhere in your code. Make sure that this is done correctly.

```csharp
public class ApplicationBase : IExternalApplication
{
	public Result OnStartup(UIControlledApplication application)
	{ 
		 ComponentManager.PreviewExecute += ComponentManager_PreviewExecute;
		 return Result.Succeeded;
	}

	private void ComponentManager_PreviewExecute(object sender, EventArgs e)
	{
		 var ButtonCaller = sender as Autodesk.Windows.RibbonButton;
		 AppGlobals.ApplicationName = ButtonCaller.Text;
	}    
}

[Autodesk.Revit.Attributes.Transaction(Autodesk.Revit.Attributes.TransactionMode.Manual)]
[Autodesk.Revit.Attributes.Regeneration(Autodesk.Revit.Attributes.RegenerationOption.Manual)]
public class TesterCommand : IExternalCommand
{
	public Result Execute(ExternalCommandData commandData, ref string message, ElementSet elements)
	{
		TaskDialog.Show("Executed Command", AppGlobals.ApplicationName);
	}
}
```