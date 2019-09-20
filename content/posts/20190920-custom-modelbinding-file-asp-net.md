+++
title = "Binding uploaded files to custom class in ASP.NET Core"
date = "2019-09-20"
author = "JK"
description = "I needed to take a file upload (IFormFile) as input in an ASP.NET Core 3 (at the time of writing, version RC1) and pass that to a backend class/processor/api in the form of a custom file class."
+++

## The problem
I needed to take a file upload (IFormFile) as input in an ASP.NET Core 3 (at the time of writing, version RC1) and pass that to a backend class/processor/api in the form of a custom file class. For purposes of this post, let's imagine that I have a backend API as follows.

```
public class Processor() {
    public async Task<bool> ProcessFile(CustomFile file)
    {
        // processing magic
        // more magic
        return true;
    }
}
```

And the CustomFile class

```
public class CustomFile {
    public CustomFile(byte[] bytes, string mimeType) {
        Bytes = bytes;
        MimeType = mimeType
    }

    public byte[] Bytes { get; }
    public string MimeType { get; }
}
```

## Solution 1 - Bind to IFormFile and translate to custom class in controller
This is the out of the box scenario with ASP.NET core.

Here's the view (using Razor pages)
```
@page
@model ProcessModel
@{
    ViewData["Title"] = "Process item";
}
<h1>Process item</h1>
<form class="ui large form" method="post" enctype="multipart/form-data">
    <div class="field">
        <label asp-for="Document"></label>
        <div class="ui input">
            <input asp-for="Document"/>
        </div>
        <div>
            <span asp-validation-for="Document" class="ui red text"></span>
        </div>
    </div>
    <input type="submit" class="ui large fluid primary button" value="Create item"/>
</form>
```

Here's your code-behind/controller
```
// usings
namespace Web.Pages.Items
{
    public class ProcessModel : PageModel
    {
        private readonly Processor _processor;

        public CreateModel(Processor processor)
        {
            _processor = processor
        }

        [BindProperty]
        public IFormFile Document { get; set; }

        public void OnGet()
        {}

        public async Task<IActionResult> OnPost()
        {
            if(!ModelState.IsValid) return Page();

            using(var ms = new MemoryStream)
            {
                await InputModel.Document.CopyToAsync(ms);
                var bytes = ms.ToByteArray();
                var customFile = new CustomFile(bytes, InputModel.Document.ContentType);
                var result = await _processor.ProcessFile(customFile);

                if(result)
                {
                    return Page("ItemProcessed");
                }
                else {
                    return Page("ItemProcessingFailed);
                }
            }
            return Page();
        }
    }
}
```

This approach gets the job done, and is a perfectly valid way of approaching the problem. However, I find this problematic for one major reason - this plumbing code will have to be repeated for every instance where we want to bind a file. Approach two addresses that.

## Solution 2
Here we use the custom model binder support in ASP.NET Core to do the plumbing only once, and reuse that everywhere that we need it. 

From the [docs](https://docs.microsoft.com/en-us/aspnet/core/mvc/advanced/custom-model-binding?view=aspnetcore-3.0)

>Model binding allows controller actions to work directly with model types (passed in as method arguments), rather than HTTP requests. Mapping between incoming request data and application models is handled by model binders. Developers can extend the built-in model binding functionality by implementing custom model binders (though typically, you don't need to write your own provider).

### The steps
1. #### Define a model binder that implements Microsoft.AspNetCore.Mvc.ModelBinding.IModelBinding

    This takes the raw HTTP request, pulls out the data of interest, and converts it into the required form. In our case, it will look at the Form files and then convert them into our CustomFile class.

    ```

    // usings
    using Microsoft.AspNetCore.Mvc.ModelBinding;
    public class CustomFileModelBinder : IModelBinder
    {
        public async Task BindModelAsync(ModelBindingContext bindingContext)
        {
            if (bindingContext == null) throw new ArgumentNullException(nameof(bindingContext));

            // read the request for any files
            var formFiles = bindingContext.ActionContext?.HttpContext?.Request?.Form?.Files;

            if (formFiles == null || !formFiles.Any())
            {
                return; // this model binder is no longer interested in this request
            }
            if (formFiles.Count == 1)
            {
                // our original plumbing logic to change IFormFiles to CustomFile class
                using (var memoryStream = new MemoryStream())
                {
                    var ff = formFiles.First();
                    await ff.CopyToAsync(memoryStream);
                    var bytes = memoryStream.ToArray();
                    var model = new CustomFile(bytes, ff.ContentType);
                    bindingContext.Result = ModelBindingResult.Success(model); // signal that model binding was successful
                }
            }
            else
            {
                // handle the case of multiple file uploads, i.e an IFormFileCollection
                var list = new List<DsdFile>();
                foreach (var formFile in formFiles)
                {
                    using (var memoryStream = new MemoryStream())
                    {
                        await formFile.CopyToAsync(memoryStream);
                        var bytes = memoryStream.ToArray();
                        var item = GetDsdFile(bytes, formFile.ContentType);
                        list.Add(item);
                    }
                }
                bindingContext.Result = ModelBindingResult.Success(list.AsEnumerable()); // signal that model binding was successful
            }
        }
    }
    ```
2. #### Define a model binder provider that implements Microsoft.AspNetCore.Mvc.ModelBinding.IModelBinderProvider
    The IModelBinderProvider determines whether or not to use a specific model binder based on the types we declared that we would like to receive in our controllers and/or razor pages. In this case, we want a CustomFile class.

    ```
    // usings
    using Microsoft.AspNetCore.Mvc.ModelBinding; 
    public class CustomFileModelBinderProvider : IModelBinderProvider
    {
        public IModelBinder GetBinder(ModelBinderProviderContext context)
        {
            if (context == null)
            {
                throw new ArgumentNullException(nameof(context));
            }

            if (context.Metadata.ModelType == typeof(CustomFile)) // note the CustomFile
            {
                return new BinderTypeModelBinder(typeof(CustomFileModelBinder));
            }

            if (typeof(IEnumerable<CustomFile>).IsAssignableFrom(context.Metadata.ModelType)) // here we want an IEnumerable of Custom File
            {
                return new BinderTypeModelBinder(typeof(CustomFileModelBinder));
            }

            return null; // our provider can't help here
        }
    }
    ```

3. #### Wire up your custom binders into ASP.NET core in Startup.cs
    We need to make the ASP.NET core pipeline aware of our new model binder. This we do in our Startup.cs file.

    ```
    public class Startup
    {
        public Startup(IConfiguration configurationd)
        {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        public void ConfigureServices(IServiceCollection services)
        {
            // ...
            services.AddMvc(options =>
                {
                    options.ModelBinderProviders.Insert(0, new CustomFileModelBinderProvider());
                });
            // ...
        }

        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            // ...
        }
    }
    ```

4. ### Adjust your controllers/handlers to bind directly to CustomFile
    Now all the logic related to mapping from an IFormFile to CustomFile is gone, and all other related controllers would look similar.

    ```
    // usings
    namespace Web.Pages.Items
    {
        public class ProcessModel : PageModel
        {
            private readonly Processor _processor;

            public CreateModel(Processor processor)
            {
                _processor = processor
            }

            [BindProperty]
            public CustomFile Document { get; set; }

            public void OnGet()
            {}

            public async Task<IActionResult> OnPost()
            {
                if(!ModelState.IsValid) return Page();

                var result = await _processor.ProcessFile(customFile);

                if(result)
                {
                    return Page("ItemProcessed");
                }
                else {
                    return Page("ItemProcessingFailed);
                }
                return Page();
            }
        }
    }
    ```

## Gotchas
- The built-in tag helpers know to display a file picker input in the browser when attempting to bind to an IFormFile. Either remember to add a "type=file" to the necessary input tags, or extend the built-in tags to become aware of the CustomFile class.
- This solution doesn't work for nested CustomFile class. For example, it will not work for a custom file that is part of an IdentificationDocument class, that itself is part of a User class.

## Further reading
- [https://stackoverflow.com/a/46124583] (https://stackoverflow.com/a/46124583)
- [https://docs.microsoft.com/en-us/aspnet/core/mvc/advanced/custom-model-binding?view=aspnetcore-3.0] (https://docs.microsoft.com/en-us/aspnet/core/mvc/advanced/custom-model-binding?view=aspnetcore-3.0)