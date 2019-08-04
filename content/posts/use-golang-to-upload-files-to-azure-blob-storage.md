---
title: "Use Golang to Upload Files to Azure Blob Storage"
date: 2019-08-04T17:33:01+07:00
draft: false
---

If you are enjoying Go and its community as much as myself but have been using the Azure Cloud Platform for some time already, you will probably want to use Azure Blob Storage at some point. In that case, this blog post is for you.

### You will need:
The first thing you need to do is make sure you set up your **blob storage account** in Azure Portal and create a **blob container**. In this example I created a blob container called *blog-photos*. Note your **blob account name** and **blob container name** because you will need these. 

<p>Your Azure Blob Service endpoint will have the structure: <b>https://<<yourblobaccountname>>.blob.core.windows.net/</b>.</p>

Your **Azure Access Key** can be found under Blob Storage Account > Access Keys. Grab the first Key and that will be sufficient. 

### Das Code:
First, let's create a method that gives us all the credentials we need for uploading files. In production you will want to use a more secure method of handling credentials, but for this example hard-coding will do. 

{{< highlight go >}}
func GetAccountInfo() (string, string, string, string) {
	azrKey := "<your_azure_access_key>"
	azrBlobAccountName := "mytechblog"
	azrPrimaryBlobServiceEndpoint := fmt.Sprintf("https://%s.blob.core.windows.net/", azrBlobAccountName)
	azrBlobContainer := "blog-photos"

	return azrKey, azrBlobAccountName, azrPrimaryBlobServiceEndpoint, azrBlobContainer
}
{{< / highlight >}}

Next, let's come up with a file name structure for our blobs. Since my sample app will upload all jpeg images in a folder, this small method will do the trick for me; a combination of date and  UUID. (The code below uses the package [github.com/gofrs/uuid](github.com/gofrs/uuid), you will need to install it to use uuid.NewV4())

{{< highlight go >}}
func GetBlobName() string {
	t := time.Now()
	uuid, _ := uuid.NewV4()

	return fmt.Sprintf("%s-%v.jpg", t.Format("20060102"), uuid)
}
{{< / highlight >}}

Finally, let's build the Uploader. This will require Azure Blob SDK: [github.com/Azure/azure-storage-blob-go/azblob](github.com/Azure/azure-storage-blob-go/azblob). 

{{< highlight go >}}
// The below method assumes you already have the byte array ready to go
func UploadBytesToBlob(b []byte) (string, error) {
	azrKey, accountName, endPoint, container := GetAccountInfo()    // This is our account info method
	u, _ := url.Parse(fmt.Sprint(endPoint, container, "/", GetBlobName()))  // This uses our Blob Name Generator to create individual blob urls
	credential, errC := azblob.NewSharedKeyCredential(accountName, azrKey)  // Finally we create the credentials object required by the uploader
	if errC != nil {
		return "", errC
	}

	blockBlobUrl := azblob.NewBlockBlobURL(*u, azblob.NewPipeline(credential, azblob.PipelineOptions{}))    // Another Azure Specific object, which combines our generated URL and credentials

	ctx := context.Background() // We create an empty context (https://golang.org/pkg/context/#Background)

    // Provide any needed options to UploadToBlockBlobOptions (https://godoc.org/github.com/Azure/azure-storage-blob-go/azblob#UploadToBlockBlobOptions)
	o := azblob.UploadToBlockBlobOptions{
		BlobHTTPHeaders: azblob.BlobHTTPHeaders{
			ContentType: "image/jpg",   //  Add any needed headers here
		},
	}

	_, errU := azblob.UploadBufferToBlockBlob(ctx, b, blockBlobUrl, o)  // Combine all the pieces and perform the upload using UploadBufferToBlockBlob (https://godoc.org/github.com/Azure/azure-storage-blob-go/azblob#UploadBufferToBlockBlob)
	return blockBlobUrl.String(), errU
}
{{< / highlight >}}

You can see a fully working example in [Github](). 