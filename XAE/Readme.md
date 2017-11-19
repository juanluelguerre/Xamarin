**Simple Xamarin Azure Explorer (XAE)**

XAE es un pequeña aplicación para compartir imágenes entre dispositivos
a la par que servir de almacenamiento de las mismas en la nube. 

La aplicación ha sido desarrollada partiendo de la plantilla de Proyecto
“Master Detail” a la que se le han añadido funcionalidades tales como la
toma de fotos a partir de la cámara (usada en Xamagram) o la selección y
el almacenamiento de fotos en la nube.

1.  La primera imagen muestra la venta de ayuda o Acerca de (About):

![About](/XAE/assets/Image1.png)

1.  La segúnda mágen es la imagen representa el navegador/explorador o
    listado de imágenes:

![Browse](/XAE/assets/Image2.png)

2.  La siguiente muetra la adicion de nuevas imágenes, previa selección de
    la acción “Add” situada en la barra de menu superior.
 
![New Item](/XAE/assets/Image3.png)

1.  La siguiente, representa a la misma ventana/pantalla en la que
    el usuario acaba de seleccionar una imagen de la galería de imágenes
    del dispositivo a través de la opción “Browse”

![New Item Selected](/XAE/assets/Image4.png)

1.  La siguiente pantalla representa el detalle de una imagén
    previamente seleccionada. En la barra de menu superior puede
    apreciarse el botón “Delete” a través del cual el usuario podrá
    eliminar dicha imagen.

![Image Detail](/XAE/assets/Image5.png)

A continuación el portal de azure con la cuenta de Azure Storage y el
contenedor “xae” donde se almacenan todas las imágnes.

![Azure Storage](/XAE/assets/Image6.png)

Finalmente, el código de la clase principal que realiza la
gestión del almacenamiento en Azure Storage es el siguiente:

```

//  
// https://developer.xamarin.com/guides/xamarin-forms/cloud-services/storage/azure-storage/  
//  
using System;  
using System.Collections.Generic;  
using System.IO;  
using System.Linq;  
using System.Threading.Tasks;  
using Microsoft.WindowsAzure.Storage;  
using Microsoft.WindowsAzure.Storage.Blob;  

namespace XAE  
{  
    public class CloudStorageService : IDataStore&lt;Item&gt;  
    {  
        private const string DEFAULT\_CONTAINER\_NAME = "xae"  
            ;  
        private IEnumerable&lt;Item&gt; items;  
  
  
        public CloudStorageService()  
        {  
            items = new List&lt;Item&gt;();  
        }  
  
        public async Task&lt;bool&gt; AddItemAsync(Item item)   
        {  
            try  
            {  
                using (var stream = item.ImageFile.GetStream())  
                {  
                    await UploadFileAsync(stream, item.Name);  
                    return true;  
                }  
            }catch(Exception ex)  
            {  
                throw ex;  
            }  
        }  
  
        public Task&lt;bool&gt; UpdateItemAsync(Item item)  
        {  
            throw new NotImplementedException();  
        }  
  
        public async Task&lt;bool&gt; DeleteItemAsync(string id)   
        {  
            try  
            {  
                var container = GetContainer();  
  
                var blob = container.GetBlobReference(id);  
                return await blob.DeleteIfExistsAsync();  
            }catch   
            {  
                return false;  
            }  
        }  
  
        public async Task&lt;Item&gt; GetItemAsync(string id)   
        {  
            var container = GetContainer();  
  
            BlobContinuationToken token = null;  
  
            var result = await container.ListBlobsSegmentedAsync(token);  
            if (result.Results.Count() &gt; 0)  
            {  
                var blobs = result.Results.Cast&lt;CloudBlockBlob&gt;()  
                                  .Where(b =&gt; b.Name.Equals(id, StringComparison.CurrentCultureIgnoreCase))  
                                  .Select(b =&gt; new Item() { Name = b.Name, Description = b.Uri.AbsoluteUri });  
                return blobs.FirstOrDefault();  
            }  
            return null;  
  
        }  
  
        public async Task&lt;IEnumerable&lt;Item&gt;&gt; GetItemsAsync(bool forceRefresh = false)  
        {  
            var container = GetContainer();  
  
            var allBlobsList = new List&lt;Item&gt;();  
            BlobContinuationToken token = null;  
  
            do  
            {  
                var result = await container.ListBlobsSegmentedAsync(token);  
  
                if (result.Results.Count() &gt; 0)  
                {  
                    var blobs = result.Results.Cast&lt;CloudBlockBlob&gt;()  
                        .Select(b =&gt; new Item() { Id=b.Name, Name = b.Name, Url = b.Uri.AbsoluteUri, Description=b.Properties.Length + " bytes. Updated: " + b.Properties.LastModified.Value.ToString("dd/MM/yyyy HH:mm:ss") });  
                    allBlobsList.AddRange(blobs);  
                }  
                token = result.ContinuationToken;  
            } while (token != null);  
  
            return allBlobsList;  
        }  
  
  
        public async Task&lt;string&gt; UploadFileAsync(Stream stream, string fileName = null)  
        {  
            try  
            {  
                var container = GetContainer();  
                await container.CreateIfNotExistsAsync();  
                await container.SetPermissionsAsync(new BlobContainerPermissions  
                {  
                    PublicAccess = BlobContainerPublicAccessType.Blob  
                });  
  
                if (String.IsNullOrWhiteSpace(fileName))  
                    fileName = Guid.NewGuid().ToString();  
  
                var fileBlob = container.GetBlockBlobReference(fileName);  
  
                //if (String.IsNullOrWhiteSpace(fileExtension))  
                //    fileExtension = DEFAULT\_EXTENSION\_FILE;  
                //fileBlob.Properties.ContentType = $"image/{fileExtension}";  
  
                await fileBlob.UploadFromStreamAsync(stream);  
  
                return fileBlob.Uri.ToString();  
            }  
            catch (StorageException ex)  
            {  
                return null;  
            }  
        }  
  
        public async Task&lt;byte\[\]&gt; DownloadFileAsync(string name)  
        {  
            var container = GetContainer();  
  
            var blob = container.GetBlobReference(name);  
            if (await blob.ExistsAsync())  
            {  
                var metadata = blob.Metadata;  
                var props = blob.Properties;  
  
  
  
                await blob.FetchAttributesAsync();  
                byte\[\] blobBytes = new byte\[blob.Properties.Length\];  
  
                await blob.DownloadToByteArrayAsync(blobBytes, 0);  
                return blobBytes;  
            }  
            return null;  
        }  
  
        public async Task&lt;bool&gt; DeleteFileAsync(string id)  
        {  
            var container = GetContainer();  
            var blob = container.GetBlobReference(id);  
            return await blob.DeleteIfExistsAsync();  
        }  
  
  
        private CloudBlobContainer GetContainer(string containerName=DEFAULT\_CONTAINER\_NAME)  
        {  
            var account = CloudStorageAccount.Parse(GlobalSettings.STORAGE\_CONNECTION\_STRING);  
            var client = account.CreateCloudBlobClient();  
  
            var container = string.IsNullOrWhiteSpace(containerName) ?  
                                   client.GetRootContainerReference() :   
                                   client.GetContainerReference((containerName));  
  
            return container;  
        }  
  
  
        private static string GetContainerSasUri(CloudBlobContainer container)  
        {  
            //Set the expiry time and permissions for the container.  
            //In this case no start time is specified, so the shared access signature becomes valid immediately.  
            SharedAccessBlobPolicy sasConstraints = new SharedAccessBlobPolicy();  
            sasConstraints.SharedAccessExpiryTime = DateTimeOffset.UtcNow.AddHours(24);  
            sasConstraints.Permissions = SharedAccessBlobPermissions.Write | SharedAccessBlobPermissions.List;  
  
            //Generate the shared access signature on the container, setting the constraints directly on the signature.  
            string sasContainerToken = container.GetSharedAccessSignature(sasConstraints);  
  
            //Return the URI string for the container, including the SAS token.  
            return container.Uri + sasContainerToken;  
        }  
  
        private static string GetBlobSasUri(CloudBlockBlob blob)  
        {  
            //Set the expiry time and permissions for the blob.  
            //In this case the start time is specified as a few minutes in the past, to mitigate clock skew.  
            //The shared access signature will be valid immediately.  
            SharedAccessBlobPolicy sasConstraints = new SharedAccessBlobPolicy();  
            sasConstraints.SharedAccessStartTime = DateTimeOffset.UtcNow.AddMinutes(-5);  
            sasConstraints.SharedAccessExpiryTime = DateTimeOffset.UtcNow.AddHours(24);  
            sasConstraints.Permissions = SharedAccessBlobPermissions.Read | SharedAccessBlobPermissions.Write;  
  
            //Generate the shared access signature on the blob, setting the constraints directly on the signature.  
            string sasBlobToken = blob.GetSharedAccessSignature(sasConstraints);  
  
            //Return the URI string for the container, including the SAS token.  
            return blob.Uri + sasBlobToken;  
        }  
    }  
}

```
