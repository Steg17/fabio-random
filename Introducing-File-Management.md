#File Management with Azure Mobile Apps#
Whether it is storing images, videos, or documents in general, modern mobile applications often have data 
storage needs that expand into binary data, and while Azure Blob Storage provides the infrastructure that is
needed to handle that task, getting it to work requires a fair amount of custom development in order to meet real world
security and scalability requirements, and the complexity is significantly increased if support offline scenarios is required,
which is not uncommon.

##I just want to store files! How hard can it be?##
While consuming the Azure Storage SDK to read and write blobs directly from your client is a pretty trivial task, 
once you start looking at real world scenarios, the following are *some* of the questions that usually come up,
and things tend to get a little more complicated:

- Where do I store my storage credentials?
- How do I prevent unauthorized access to user files?
- How do I make sure my service availability is not impacted by file management?
- How do I persist changes when my client is offline? 
- How do I synchronize files when client connectivity is restored?

One of the goals of the Mobile Apps team is to make mobile development as easy as possible, and when it came to file management we
knew we had an opportunity to make things better.

##Introducing Azure Mobile Apps File Management SDK##

The Azure Mobile Apps File Management SDK, is a library that provides client and server components designed to help eliminate 
the complexities outlined above in application using the Azure Mobile Apps SDK, currently using Azure Blob Storage as the underlying storage infrastructure.

On the server, the SDK adds the functionality required to safely issue and manage SAS tokens, manage file associations and provide facilities such as the new `StorageController` 
class and other types to allow developers to inject custom authorization, container naming and scoping logic as well as some more advanced customization features (which will be the topic of another post).

At its core, the client SDK enables offline file synchronization, enumeration, automatic token acquisition and communication wih the underlying store.

##Getting started##

The easiest way to demonstrate how the SDK simplifies the file management process is to walk through a simple implementation, so let's do just that.

###Creating the server###

To get started, let's add a reference to the **Microsoft.Azure.Mobile.Server.Files** NuGet package to a new or existing Mobile App Server project and add a new `StorageController`:

```csharp
[RoutePrefix("tables/todoitem")]
public class TodoItemStorageController : StorageController<TodoItem>
{
    [HttpPost]
    [Route("{id}/StorageToken")]
    public async Task<IHttpActionResult> PostStorageTokenRequest(string id, StorageTokenRequest value)
    {
        StorageToken token = await GetStorageTokenAsync(id, value);

        return Ok(token);
    }

    [HttpGet]
    [Route("{id}/MobileServiceFiles")]
    public async Task<IHttpActionResult> GetFiles(string id)
    {
        IEnumerable<MobileServiceFile> files = await GetRecordFilesAsync(id);

        return Ok(files);
    }

    [HttpDelete]
    [Route("{id}/MobileServiceFiles/{name}")]
    public Task Delete(string id, string name)
    {
        return base.DeleteFileAsync(id, name);
    }
}
```     

In the current version of the File Management SDK, files are always managed within the context of a Mobile App data entity, in the example above, 
we are creating a storage controller for the well known *TodoItem*, used in the Mobile Apps quick start project.

The implementation above relies on a storage connection string to be present in in configuration. By convention, the connection string should be named
**MS_AzureStorageAccountConnectionString**, but you can use a different name by passing it into one of the constructors exposed by the `StorageController` class.

>The connection string must be a properly formatted storage connection string.
>You can find your account's connection string by opening the Azure portal and going to:

>Your Storage Account > Settings > Access Keys > Connection Strings

Now that our storage controller is defined, our service exposes the APIs needed by the client SDK, so let's look at that next.

###Client Implementation###

##Summary##
The Mobile Apps File Management SDK makes working with files in mobile applications using Azure Mobile Apps simple, without compromising on recommended practices 
when it comes to security and performance.

The project is **Open Source** and developed in the open, so if you want to track what we're working on, file issues, submit feedback or get the source code, 
just head over to our GitHub repositories at https://github.com/azure/azure-mobile-apps-net-files-client/ , for the client SDK or https://github.com/azure/azure-mobile-apps-net-files-server 
for the server.

In addition to GitHub, you can also ask questions on the Mobile Apps MSDN forum at https://social.msdn.microsoft.com/Forums/en-US/home?forum=azuremobile

We're making this early beta release available so you can get a taste of some of the file management features we're working on and to get your feedback, 
so please don't hesitate to let us know what you like and more importantly, what you don't.  