---
title: "Get UIApplication"
datePublished: Sat May 04 2024 17:58:29 GMT+0000 (Coordinated Universal Time)
cuid: clvseqnhi000109lfhsat9t9j
slug: get-uiapplication
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1715613019338/b7ae12a4-08e7-4724-a681-547117b029b1.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1715613006400/917bf9a7-c6b9-4753-9899-c2f212b078a4.png
tags: revitapi, uiapplication, how-to-get-uiapp

---

UIApplication is a crucial class for developers as it offers access to ["UI customization methods, events, the main window, and the active document."](https://www.revitapidocs.com/2019/51ca80e2-3e5f-7dd2-9d95-f210950c72ae.htm)

Fortunately, an instance of this class can be obtained from various locations. For instance:

1. When an IExternalCommand class is executed, you can retrieve it from the provided parameter ExternalCommandData. See this example.
    
    ```csharp
    public override Result Execute(ExternalCommandData commandData, ref string message, ElementSet elements)
    {
        UIApplication uiapp = commandData.Application;
        //... more code statements
    }
    ```
    

2. Another possible way to access it is when an ExternalEvent triggers an event to execute an instance of [IExternalEventHandler](https://www.revitapidocs.com/2019/dc7ccfc6-f7a2-6f52-1a14-1cb5406ebc09.htm). The implementation of this interface includes a method Execute that takes a parameter UIApplication. After executing the method, you can capture this instance.
    
    ```csharp
    public void Execute(UIApplication uiApp)
    ```
    

3. In addition to that, another way to access it is by initializing it if you already have access to the ApplicationServices.Application instance. The object itself can be retrieved using Document.Application, and then we initialize a new instance of the UI application from this object.
    
    ```csharp
    UIApplication uiapp = new UIApplication(Doc.Application); 
    ```
    

4. Also, when Revit initializes early on, refer to this code sample for a clearer illustration.
    
    ```csharp
    public virtual Result OnStartup(UIControlledApplication application)
    {
        application.ControlledApplication.ApplicationInitialized += ControlledApplication_ApplicationInitialized;
        //... more code statements
    }
    
    void ControlledApplication_ApplicationInitialized(object sender, Autodesk.Revit.DB.Events.ApplicationInitializedEventArgs e)
    {
       Autodesk.Revit.ApplicationServices.Application m_app = sender as Autodesk.Revit.ApplicationServices.Application;
       UIApplication uiApp = new UIApplication(m_app);
    }
    ```
    

5. Similarly to the Application Initialized event, we can also utilize idling and other events from the startup.
    
    ```csharp
    UIControlledApplication UcApp;
    public virtual Result OnStartup(UIControlledApplication application)
    {
        UcApp = application;    
        application.Idling += GetUiApp;
        //... more code statements
    }
    
    protected void GetUiApp(object sender, Autodesk.Revit.UI.Events.IdlingEventArgs e)
    {
        UcApp.Idling -= GetUiApp;
        UIApplication UiApp = sender as UIApplication;
        OnIdling();
    }
    ```
    

Maybe there are more ways to fetch this instance. So far, I believe all the possibilities mentioned above are sufficient to obtain a hold of this class instance.