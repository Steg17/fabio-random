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
The File Management Client SDK extends the Mobile App Client SDK with the operations needed to manage and synchronize files. For this example, let's
take a look at what is needed to enable offline file management and synchronization.

To get started add a reference to the **Microsoft.Azure.Mobile.Client.Files** package (as of this writing, make sure you're including pre-release packages), 
once the package is installed, we need to initialize the file synchronization context. This allows us to specify the store we want to use to track the file operations and store the file metadata and provide
an instance of an `IFileSyncHandler`, which is the application logic that decides how to handle file operations and how to read files when they need to be uploaded to the underlying store (more on this later).

If you're updating an existing Mobile App client, you can initialize the file synchronization context with a single line of code:

```csharp
client.InitializeFileSyncContext(new FileSyncHandler(), store);
``` 

Here's what the Mobile Apps client code looks like once updated to handle files: 

```csharp
MobileServiceClient client = new MobileServiceClient("https://<mymobileservice>.azurewebsites.net/");

var store = new MobileServiceSQLiteStore("mydatabase.db");
store.DefineTable<DataEntity>();

// Code added to initialize the file synchronization context:
client.InitializeFileSyncContext(new FileSyncHandler(), store);

client.SyncContext.InitializeAsync(store);
```

With the code above in place, we're ready to start working with files with offline support!

####Managing files####
As previously mentioned files exist within the context of a record. This means that files managed by the SDK are always
associated with a data entity, therefore all file management operations are exposed as methods on `IMobileServiceTable` and
`IMobileServiceSyncTable` when working with offline support.

With that in mind, if we want to create a new file associated with a TodoItem, we can simply call the `AddFileAsync` on the its table:
```csharp
await _todoItemTable.AddFileAsync(todoItem, "myfilename");
``` 

Where `todoItem` is an instance of a `TodoItem` entity and  `_todoItemTable` is an instance of an `IMobileServiceSyncTable<TodoItem>` obtained 
by calling `GetSyncTable` as you normally would in your Mobile App client:
```csharp
// _todoItemTable is a private class variable of type IMobileServiceSyncTable<TodoItem>
_todoItemTable = client.GetSyncTable<TodoItem>();
```

To delete a file, you can call the `DeleteFileAsync` method on the table:
```csharp
await _todoItemTable.DeleteFileAsync(myfile);
```

To enumerate files associated with a given data entity, you can simply call the `GetFilesAsync` method:
```csharp
IEnumerable<MobileServiceFile> files = await _todoItemTable.GetFilesAsync(todoItem);
```

It's important to understand that all of the operations described above are working completely offline. There are no network
calls or data being perstisted to the target storage infrastructure (Azure Blob Storage) when those calls are made.

####Synchronizing files####
Similar to the way you synchronize table data using the Azure Mobile App Client SDK, in order to persist file changes (upload/delete files), you can 
simply call the  `PushFileChangesAsync` method on your table:
```csharp
await _todoItemTable.PushFileChangesAsync();
```

Calling this method will begin the process of processing local file operations and taking the appropriate actions against the target store (Azure Blob Storage), this includes
handling token acquisition when needed to make sure all operations are properly authenticated.

Pulling file changes from the server uses a process based on synchronization triggers. We won't delve into the details of how synchronization triggers work (another topic for another post), 
but with the default trigger that comes "out of the box", the file synchronization takes place as part of the standard data synchronization process that is part of the Azure Mobile Apps Client SDK,
so calling `PullAsync` on your sync table will retrieve all file operations that have taken place since your last synchronization for those entities and initiate the synchronization process.

```csharp
await _todoItemTable.PullAsync("todoitems", string.Empty);
```    

###The IFileSyncHandler###
As promised, let's now get back to the `IFileSyncHandler` object. 

After going through the examples above, the more observant readers are probably wondering where files are saved when downloaded and read from when uploaded to the
target store. Well, my friends, that is where the `IFileSyncHandler` comes into play.

As previously mentioned, the concrete implementation of an `IFileSyncHandler` is **part of your application logic**, that means that utimately, you decide where, how and **if** files should be
stored on the local device and, as a result, the SDK also asks the synchronization handler for a file data reference when it actually needs to read a file (as in the case of an upload).

With two methods, the `IFileSyncHandler` is a simple interface to implement, but it has an important job and gives you the flexibility to decide how to handle file operations.

Let's take a look at a sample implementation of a file synchronization handler:

```csharp
public class FileSyncHandler : IFileSyncHandler
{
    private readonly IMobileServiceSyncTable<TodoItem> _todoItemTable;

    public FileSyncHandler(IMobileServiceSyncTable<TodoItem> todoItemTable)
    {
        _todoItemTable = todoItemTable;
    }

    public async Task<IMobileServiceFileDataSource> GetDataSource(MobileServiceFileMetadata metadata)
    {
        string filePath = ResolvePathForMyApplication(metadata.ParentDataItemType, metadata.ParentDataItemId);

        return new PathMobileServiceFileDataSource(filePath);
    }

    public async Task ProcessFileSynchronizationAction(MobileServiceFile file, FileSynchronizationAction action)
    {
        if (action == FileSynchronizationAction.Create || action == FileSynchronizationAction.Update)
        {
            // Look for user defined metadata indicating that this file should be automatically downloaded and
            // available offline:
            string persistOfflineValue;
            if (file.Metadata.TryGetValue("AvailableOffline", out persistOfflineValue)
                && string.Compare(persistOfflineValue, "true", StringComparison.OrdinalIgnoreCase) == 0)
            {
                // File should be available offline, resolve the path and download it:
                string filePath = ResolvePathForMyApplication(file.TableName, file.ParentId);
                await _todoItemTable.DownloadFileAsync(file, filePath);
            }
        }
        else if (action == FileSynchronizationAction.Delete)
        {
            // Delete file if it exists...
        }
   }

    private string ResolvePathForMyApplication(string parentDataItemType, string parentDataItemId)
    {
        ...
        // custom application logic to decide where files should be stored.
        ...
    }
}
```

In the example above, we have a simple file sync handler that conditionally downloads files when they become available (or are updated), 
based on a user defined property (which is fully controlled by the application code), essentially deciding that only some of the files, not all, should be available
if the device is taken offline. As you can see, the handler also decides where those files should be persisted and read from.

##Summary##
The Mobile Apps File Management SDK makes working with files in mobile applications using Azure Mobile Apps simple, without compromising on recommended practices 
when it comes to security and performance.

The project is **Open Source** and developed in the open, so if you want to track what we're working on, file issues, submit feedback or get the source code, 
just head over to our GitHub repositories at https://github.com/azure/azure-mobile-apps-net-files-client/ , for the client SDK or https://github.com/azure/azure-mobile-apps-net-files-server 
for the server.

In addition to GitHub, you can also ask questions on the Mobile Apps MSDN forum at https://social.msdn.microsoft.com/Forums/en-US/home?forum=azuremobile

We're making this early beta release available so you can get a taste of some of the file management features we're working on and to get your feedback, 
so please don't hesitate to let us know what you like and more importantly, what you don't.  