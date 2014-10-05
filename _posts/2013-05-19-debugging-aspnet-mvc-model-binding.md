---
layout: post
title: "Debugging ASP.NET MVC Model Binding"
description: ""
category: csharp
tags: [aspnetmvc,csharp]
---
{% include JB/setup %}

## The Scenario ##

Recently we had a tricky defect raised where, under certain circumstances, the user-entered values were not being persisted. The page was a somewhat complex data entry form, with many fields, lots of validation, and a complex hierarchy of view model objects and controls/helpers.

## The process ##

After reproducing the issue (which itself was difficult), the first thing we did was to attach the debugger and set a break point on the first line of the controller action.

{% highlight c# %}
[HttpPost]
public void SubmitAction(ComplexModel model, IList<MoreComplex> more)
{
	if (!ValidForSubmit(model, more)) {
		return RepopulateAndView(model, more);
	}
	_service.Save(model, more);
	return RedirectToAction("Index");
}
{% endhighlight %}

### Attach a debugger ###

Upon breaking into the debugger, we noticed the `ComplexModel` was missing certain values that were entered into the form. After checking the Razor template, we couldn't see any obvious issues, so we used IE and Chrome developer tools to inspect the running page. Again, no obvious issues or explanations.

### Inspect the request variables ###

The next step was to check the `Request.Form` variables. Strange... the correct values had been submitted, and they were structured and named correctly. Instead of trusting ASP.NET MVC entirely, we also attached [Fiddler](<http://fiddler2.com>). Once again, the missing values appeared and looked correct! Still no explanation for the missing values within our ```ComplexModel```.

### Debug the Framework ###

At this point, we guessed there was a problem in the [Default Model Binder](<http://msdn.microsoft.com/en-us/library/system.web.mvc.defaultmodelbinder.aspx>), however, we remained skeptical. The best option might be to debug into the `DefaultModelBinder`, and try to figure out why it's not binding the missing fields.

Unfortunately, we hit numerous issues at this step. We found some [great suggestions](<http://stackoverflow.com/a/4651176>) on [StackOverflow](<http://stackoverflow.com>), which unfortunately didn't work as hoped. Our developer PCs were locked down in an [SOE](<http://en.wikipedia.org/wiki/Standard_Operating_Environment>) which meant we didn't have the necessary configuration and permissions to download from the source server and [Debug the framework](<http://weblogs.asp.net/gunnarpeipman/archive/2010/07/04/stepping-into-asp-net-mvc-source-code-with-visual-studio-debugger.aspx>).

Instead we tried downloading the [MVC Source](<http://aspnetwebstack.codeplex.com/>) and [copy it into our solution](<http://blog.stevensanderson.com/2009/02/03/using-the-aspnet-mvc-source-code-to-debug-your-app/>). Unfortunately, the project configuration was again somewhat complex, with configuration files, addins, third part libraries, all using the strongly signed ASP.NET MVC assemblies. After a few hours of trouble, we decided to try another option.

### Replace the DefaultModelBinder ###

We grabbed a copy of the [DefaultModelBinder](<https://github.com/ASP-NET-MVC/aspnetwebstack/blob/v2-rtm/src/System.Web.Mvc/DefaultModelBinder.cs>), copied it into our project, renamed it to `DebugDefaultModelBinder`, fixed a few compile errors where it was referencing MVC internal classes and methods. 

In our [Application_Start](<http://bubblogging.wordpress.com/2013/01/12/mvc-4-application-start/>) method, we changed the application to use this new model binder: `ModelBinders.Binders.DefaultBinder = new DebugDefaultModelBinder();`.

### Create a simplified test case ###

It was incredibly time consuming and error prone to continue loading up the application, navigating to the correct form, filling out all the fields so client-side validation would succeed, attaching the debugger, then pressing submit.

Instead, we looked for ways to recreate the failing scenario in a unit test. We found an [old post](<http://www.hanselman.com/blog/SplittingDateTimeUnitTestingASPNETMVCCustomModelBinders.aspx>) describing how to achieve this. Basically, create a unit test like this:

{% highlight csharp %}
[TestClass]
public class ModelBinderShould
{
	private class ComplexModel
	{
	    public int? ChildrenId { get; set; }
	    public virtual IList<ComplexModelChild> Children { get; set; }
	    public string Field1 { get; set; }
	}
	
	private class ComplexModelChild
	{
	    public int Id { get; set; }
	    public int? Value { get; set; }
	}
	
	[TestMethod]
	public void BindOurChildrenModelFieldsCorrectly()
	{
	    // arrange
	    var bindingContext = new ModelBindingContext
	    {
	        ValueProvider = new NameValueCollectionValueProvider(new NameValueCollection
	        {
	            {"ChildrenId1", "1"},
	            {"ChildrenId2", "2"},
	            {"ChildrenId3", "3"},
	            {"Children[0].Id", "0"},
	            {"Children[0].Value", "10"},
	            {"Field1", "Value1"},
	        }, CultureInfo.CreateSpecificCulture("en-AU")),
	        ModelMetadata = ModelMetadataProviders.Current.GetMetadataForType(null, typeof (ComplexModel)),
	    };
	
	    var binder = new DebugDefaultModelBinder();
	
	    // Act
	    var result = (ComplexModel) binder.BindModel(new ControllerContext(), bindingContext);
	
	    // Assert
	    Assert.IsNotNull(result.Children);
	    Assert.AreEqual(1, result.Children.Count());
	}
}
{% endhighlight %}

### Diving into the DefaultModelBinder ###

Now that we have a relatively targeted test case, debugging the problem should be a lot easier. We step into our `DebugDefaultModelBinder` and notice the following:

![Debug Model Binder Problem Found](https://raw.github.com/adamchester/test/master/debugging-aspnet-model-binding-problem.png "Debug Model Binder Problem Found")

At this point, we realise that our `Children` property is being skipped because the `BindingContext.ValueProvider.ContainsPrefix` method returns `false`!

### Finding the real bug ###

The following test case now gets us closer to the real problem:

{% highlight C# %}
[TestMethod]
public void ValueProvider_ShouldContainGoPrefix()
{
    var valueProvider = new NameValueCollectionValueProvider(new NameValueCollection
    {
        {"GoX", "1"},
        {"Go[0].Id", "0"},
        {"Field1", "1"},
    }, CultureInfo.CreateSpecificCulture("en-AU"));

    Assert.IsTrue(valueProvider.ContainsPrefix("Go"));
}
{% endhighlight %}

It fails! We seem to have found the bug, and head back to [google](<http://www.google.com/?q=mvc+bug+value+provider+contains+prefix>) where we find out more:

The bug has been reported a number of times:
 1. [FormValueProvider skips form values](<http://forums.asp.net/t/1856357.aspx/1?FormValueProvider+skips+form+values>)
 2. And directly reported on codeplex [616](<http://aspnetwebstack.codeplex.com/workitem/616>) and [832](<http://aspnetwebstack.codeplex.com/workitem/832>) at least twice.

The bug has been [fixed](<https://aspnetwebstack.codeplex.com/SourceControl/changeset/a3bb0ba79317584d8a0eec1467bbfe95fa7f580d>), but not yet released.

The person who eventually fixed the bug, commented:

> If there's something in the search boundary that starts with the same name as the collection prefix that we're trying to find, the binary search would actually fail. For example, let's say we have foo.a, foo.bM and foo.b[0]. Calling Array.BinarySearch will fail to find foo.b because it will land on foo.bM, then look at foo.a and finally failing to find the prefix which is actually present in the container (foo.b[0]). Here we're doing another pass looking specifically for collection prefix.

A number of work-arounds exist:
 1. Simple: avoid having a child collection with the same `prefix` text that other fields begin with.
 2. More complex: [Fixing the Model Binding issue of ASP.NET MVC 4 and ASP.NET Web API](<http://weblogs.asp.net/imranbaloch/archive/2012/12/08/fixing-model-the-binding-issue-of-asp-net-mvc-and-asp-net-web-api.aspx>)
