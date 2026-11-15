# How to Migrate from AWS S3 to DigitalOcean Spaces: A Complete Step-by-Step Guide

## Introduction

Cloud object storage plays a critical role in modern web applications. It is commonly used to store images, videos, documents, user-uploaded files, backups, and other static assets.

**Amazon Web Services (AWS) S3** is one of the most widely adopted object storage solutions. However, depending on application requirements, infrastructure costs, operational simplicity, and CDN integration, developers may consider alternative storage providers.

One such alternative is **DigitalOcean Spaces**, an S3-compatible object storage service that provides built-in CDN integration.

Migrating from AWS S3 to DigitalOcean Spaces does not necessarily require rewriting an application's entire storage architecture. Since DigitalOcean Spaces supports the S3 API, existing applications using S3-compatible SDKs can often be adapted with relatively minor changes.

This guide explains the complete migration process, covering storage configuration, asset transfer, backend integration, frontend updates, database URL migration, production deployment, and verification.

---

## 1. Understanding the Migration Architecture

Before beginning the migration, it is important to understand how both storage services operate.

AWS S3 and DigitalOcean Spaces provide object storage through buckets. Files are stored as objects and accessed through API endpoints or publicly accessible URLs, depending on the configured permissions.

DigitalOcean Spaces supports the S3-compatible API, allowing developers to reuse familiar tools and SDKs.

### Key differences

| Component | AWS S3 | DigitalOcean Spaces |
|---|---|---|
| Storage service | Amazon S3 | DigitalOcean Spaces |
| API compatibility | S3 API | S3-compatible API |
| Storage structure | Buckets and objects | Spaces and objects |
| SDK compatibility | AWS SDK | AWS SDK with custom endpoint |
| CDN | CloudFront or other CDN | Built-in Spaces CDN |
| Authentication | AWS credentials | Spaces access keys |

### Recommended migration strategy

A structured migration should follow these stages:

1. Create and configure the destination storage.
2. Configure access credentials.
3. Establish connections to both storage providers.
4. Transfer existing files.
5. Verify file integrity.
6. Update backend upload and deletion services.
7. Update frontend image configurations.
8. Migrate existing database references.
9. Deploy the updated application.
10. Monitor and validate production behavior.

The objective is to migrate storage without disrupting existing application functionality.

---

## 2. Setting Up DigitalOcean Spaces

The first step is to create a destination storage environment.

### Step 1: Create a Space

Log in to the DigitalOcean control panel and navigate to the Spaces section.

Create a new Space with the following configuration:

- **Region:** Select a region geographically suitable for your application's users.
- **Space Name:** Choose a unique storage name.
- **CDN:** Enable CDN integration if your application serves public static assets.
- **Permissions:** Configure access according to your application's security requirements.

Choosing an appropriate region can help reduce network latency between your application infrastructure and storage service.

### Step 2: Generate Access Keys

Navigate to the API settings and generate Spaces access keys.

You will receive two credentials:

- Access Key ID
- Secret Access Key

These credentials allow applications and migration tools to communicate with DigitalOcean Spaces.

**Security recommendation:** Never expose storage credentials in frontend code, public repositories, or client-side environment variables. Store them securely on the backend.

### Step 3: Understand the Storage Endpoint

DigitalOcean Spaces provides region-specific endpoints.

Applications use these endpoints to communicate with the storage API, while CDN-enabled Spaces can provide a separate URL for delivering publicly accessible assets.

Keep API endpoints and public CDN endpoints separate in your application configuration.

---

## 3. Installing and Configuring rclone

Transferring files manually is inefficient, especially when dealing with thousands of objects.

For this migration, we can use **rclone**, an open-source command-line utility that supports multiple cloud storage providers and can transfer files directly between them.

One of its biggest advantages is that files can be transferred between remote storage services without manually downloading every object to your local machine.

### Step 1: Install rclone

On Windows, install using:

```powershell
winget install Rclone.Rclone
```

On Ubuntu or Debian-based Linux distributions:

```bash
sudo apt update
sudo apt install rclone
```

Alternatively, use the official installation method appropriate for your operating system.

Verify the installation:

```bash
rclone --version
```

### Step 2: Configure both storage providers

Run:

```bash
rclone config
```

Create two remote configurations:

- One for AWS S3.
- One for DigitalOcean Spaces.

The configuration should contain the respective provider types, access credentials, region information, and endpoints.

An example configuration structure:

```ini
[aws]
type = s3
provider = AWS
access_key_id = AWS_ACCESS_KEY
secret_access_key = AWS_SECRET_KEY
region = AWS_REGION

[digitalocean]
type = s3
provider = DigitalOcean
access_key_id = DO_ACCESS_KEY
secret_access_key = DO_SECRET_KEY
endpoint = https://REGION.digitaloceanspaces.com
region = REGION
```

Replace the placeholder values with credentials and configuration values from your respective cloud accounts.

### Step 3: Verify both connections

Before transferring any data, verify that rclone can communicate with both providers.

```bash
rclone lsd aws:
```

```bash
rclone lsd digitalocean:
```

These commands should list the available storage buckets or Spaces.

If the connections fail, verify the credentials, endpoint, region, and network connectivity before continuing.

---

## 4. Migrating Existing Storage Assets

Once both connections are working, the next step is transferring existing files.

### Step 1: Copy files from AWS S3

Use the following command:

```bash
rclone copy aws:SOURCE_BUCKET digitalocean:DESTINATION_SPACE --progress
```

This command transfers files from the source bucket to the destination Space while preserving the directory structure.

For applications serving publicly accessible media, configure the appropriate destination object permissions.

For example, where public access is intentionally required:

```bash
rclone copy aws:SOURCE_BUCKET digitalocean:DESTINATION_SPACE --s3-acl=public-read --progress
```

**Important:** Do not enable public-read permissions for private documents, sensitive user files, or confidential application data. Use private storage and controlled access mechanisms where necessary.

### Step 2: Verify file integrity

After transferring the assets, compare the source and destination.

```bash
rclone check aws:SOURCE_BUCKET digitalocean:DESTINATION_SPACE --one-way
```

This helps identify missing or different files in the destination.

A successful verification should report no differences.

For critical migrations, also compare object counts, sizes, and checksums where supported.

### Step 3: Transfer selected local assets

Sometimes an application contains additional static assets stored locally rather than in cloud storage.

You can transfer a specific directory using:

```bash
rclone copy ./public/assets digitalocean:DESTINATION_SPACE/assets --progress
```

To exclude particular folders:

```bash
rclone copy ./public/assets digitalocean:DESTINATION_SPACE/assets --exclude "**/excluded-folder/**" --progress
```

This is useful when only selected directories need to be migrated.

### Step 4: Test destination file accessibility

Once the transfer is complete, test representative files using HTTP requests.

```bash
curl -I "https://YOUR_CDN_ENDPOINT/path/to/file.jpg"
```

A successful response, such as HTTP 200, indicates that the requested object is accessible through the endpoint.

Also test different file types, including images, PDFs, and other assets used by the application.

---

## 5. Updating Backend Storage Configuration

Transferring files is only one part of the migration.

The backend must also be configured to upload new files, delete existing files, and generate valid URLs using DigitalOcean Spaces.

Since Spaces supports the S3 API, applications using the AWS SDK can often continue using the same SDK package.

### Step 1: Install or reuse the AWS SDK

For Node.js applications:

```bash
npm install @aws-sdk/client-s3
```

If the application already uses this package, no additional installation is required.

### Step 2: Configure environment variables

Update the backend environment configuration:

```env
DOS_ACCESS_KEY_ID=your_access_key
DOS_SECRET_ACCESS_KEY=your_secret_key

DOS_REGION=your_region
DOS_ENDPOINT=https://your_region.digitaloceanspaces.com

DOS_BUCKET_NAME=your_space_name
DOS_BASE_FOLDER=
```

Environment variables allow storage configuration to remain separate from application logic.

Never commit actual credentials to version control.

### Step 3: Configure the S3-compatible client

Create a storage client using the AWS SDK:

```javascript
const {
  S3Client,
  PutObjectCommand,
  DeleteObjectCommand,
} = require("@aws-sdk/client-s3");

const dos = new S3Client({
  region: process.env.DOS_REGION,

  endpoint: process.env.DOS_ENDPOINT,

  credentials: {
    accessKeyId: process.env.DOS_ACCESS_KEY_ID,
    secretAccessKey: process.env.DOS_SECRET_ACCESS_KEY,
  },
});
```

The important difference is the custom endpoint and DigitalOcean Spaces credentials.

The SDK continues to provide familiar S3 operations.

### Step 4: Implement file uploads

Use the `PutObjectCommand` to upload files.

```javascript
const uploadFile = async (file, fileName) => {
  try {
    const params = {
      Bucket: process.env.DOS_BUCKET_NAME,
      Key: fileName,
      Body: file.buffer,
      ContentType: file.mimetype,
    };

    await dos.send(new PutObjectCommand(params));

    return true;
  } catch (error) {
    console.error("File upload failed:", error);
    return false;
  }
};
```

For public media, configure the appropriate access policy and generate the corresponding CDN URL.

### Step 5: Implement file deletion

Existing applications frequently store uploaded file URLs in their database.

When a file is deleted, the backend should extract the object key and send a deletion request to the destination storage provider.

```javascript
const deleteFile = async (fileName) => {
  try {
    await dos.send(
      new DeleteObjectCommand({
        Bucket: process.env.DOS_BUCKET_NAME,
        Key: fileName,
      })
    );

    return true;
  } catch (error) {
    console.error("File deletion failed:", error);
    return false;
  }
};
```

Keeping upload and deletion operations inside a dedicated storage service makes future provider migrations easier.

### Step 6: Preserve existing function compatibility

If multiple controllers already depend on an existing storage utility, avoid unnecessarily changing every controller.

Instead, maintain the existing function interfaces and update their internal implementation.

This approach reduces code changes and minimizes the possibility of introducing bugs during migration.

---

## 6. Updating Frontend and Dashboard Configuration

After changing the backend storage provider, the frontend must also recognize the new image domains.

This is particularly important for applications built with Next.js.

Next.js restricts external image sources through its image configuration.

### Step 1: Update Next.js remote image patterns

In the frontend's Next.js configuration:

```javascript
const nextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: "https",
        hostname: "**.digitaloceanspaces.com",
      },
    ],
  },
};

module.exports = nextConfig;
```

This allows images served through supported DigitalOcean Spaces hostnames.

If the application uses a separate CDN hostname, configure that hostname explicitly as well.

Apply the same configuration to other frontend applications or administrative dashboards that display remotely hosted images.

### Step 2: Replace hardcoded storage URLs

Search the application for hardcoded AWS S3 URLs.

Update references to point to the new storage or CDN endpoint.

However, avoid blindly replacing every occurrence of an AWS hostname. Some references may belong to unrelated integrations or files that have not yet been migrated.

A better approach is to centralize storage URL generation:

```javascript
const getAssetUrl = (filePath) => {
  return `${process.env.NEXT_PUBLIC_ASSET_BASE_URL}/${filePath}`;
};
```

The frontend can then use a configurable asset base URL instead of depending on a hardcoded storage provider.

Only public CDN endpoints should be exposed through frontend environment variables.

### Step 3: Rebuild the frontend

Changes to Next.js configuration generally require a new production build.

```bash
npm run build
```

Restart the application after deploying the updated build.

---

## 7. Migrating Existing Database URLs

This is one of the most important steps in the entire migration.

Even after successfully transferring all files, your application may continue displaying broken images because existing database records still contain AWS S3 URLs.

For example, a database may contain storage URLs inside:

- User profile images.
- Blog cover images.
- Embedded HTML content.
- Product images.
- Document references.
- Other application-specific records.

Simply migrating the physical files will not update these references.

### Step 1: Identify affected database records

Search the database for records containing the old storage hostname.

For MongoDB, a simple query might look like:

```javascript
db.users.find({
  image: {
    $regex: "amazonaws.com"
  }
});
```

Perform similar checks for other collections that store file URLs.

### Step 2: Replace old URLs with new URLs

For a MongoDB-based application, the following example demonstrates how URL references can be updated.

```javascript
const mongoose = require("mongoose");

const OLD_BASE_URL = "OLD_STORAGE_BASE_URL";
const NEW_BASE_URL = "NEW_STORAGE_BASE_URL";

async function migrateUrls() {
  try {
    await mongoose.connect(process.env.MONGO_URI);

    const db = mongoose.connection.db;

    const collections = ["users", "blogs"];

    for (const collectionName of collections) {
      const collection = db.collection(collectionName);

      const records = await collection
        .find({
          image: {
            $regex: OLD_BASE_URL,
          },
        })
        .toArray();

      for (const record of records) {
        const updatedImage = record.image.replace(
          OLD_BASE_URL,
          NEW_BASE_URL
        );

        await collection.updateOne(
          { _id: record._id },
          {
            $set: {
              image: updatedImage,
            },
          }
        );
      }

      console.log(`Migration completed: ${collectionName}`);
    }
  } catch (error) {
    console.error("Database migration failed:", error);
  } finally {
    await mongoose.disconnect();
  }
}

migrateUrls();
```

This is a simplified example for a specific field structure.

In real applications, migration scripts should also account for:

- Nested objects.
- Arrays of image URLs.
- Embedded HTML content.
- Multiple URL formats.
- Collection-specific field names.

For large databases, consider processing records in batches instead of loading the entire collection into memory.

### Step 3: Verify database migration

After execution, search for remaining references to the old storage endpoint.

Verify that the updated URLs correctly resolve to files in the destination Space.

**Important:** Take a database backup before running bulk update operations. A migration script should be tested against a staging database before being executed in production.

---

## 8. Production Deployment

Once the storage, backend, frontend, and database changes are complete, the application can be deployed.

A controlled deployment is preferable to immediately removing the old storage configuration.

### Step 1: Commit the changes

Commit the updated application code to version control.

```bash
git add .
git commit -m "Migrate object storage to DigitalOcean Spaces"
git push
```

### Step 2: Deploy the backend

Update the production environment variables with the destination storage credentials and configuration.

Deploy the updated backend application and restart its services.

### Step 3: Deploy the frontend

Build and deploy the updated frontend.

```bash
npm install
npm run build
```

Restart the application using the relevant process manager or deployment platform.

### Step 4: Validate production functionality

Test the following operations:

1. Upload a new image.
2. Open the uploaded image through its CDN URL.
3. Display existing migrated images.
4. Delete an uploaded file.
5. Display images inside blog content.
6. Display user profile images.
7. Verify dashboard image rendering.
8. Check application logs for storage errors.

Do not decommission the original AWS storage immediately.

Keep it available until the new environment has passed the required production checks and a rollback plan is no longer needed.

---

## 9. Troubleshooting Common Migration Issues

Even when the migration process is followed correctly, certain issues can occur.

### Issue 1: Next.js reports that the image URL is not allowed

**Possible cause:**

The destination storage hostname has not been added to the Next.js `remotePatterns` configuration.

**Solution:**

Add the appropriate hostname, rebuild the frontend, and restart the application.

### Issue 2: HTTP 403 Forbidden

**Possible cause:**

The requested file is private or its access policy does not permit public access.

**Solution:**

Check the object's permissions and bucket access policy.

For public media, configure the required public access permissions. For private media, use authenticated access or signed URLs instead.

### Issue 3: Images are missing after migration

**Possible causes:**

- Some objects were not transferred.
- Database URLs still reference the old provider.
- The destination object keys differ from the original keys.
- The frontend is using an outdated URL.

**Solution:**

Run the storage comparison command again, inspect database references, and verify the destination object paths.

### Issue 4: Old images continue appearing

**Possible cause:**

The CDN or browser is serving cached content.

**Solution:**

Purge the relevant CDN cache when necessary and verify cache-control settings.

### Issue 5: Upload operations fail

**Possible causes:**

- Incorrect access credentials.
- Incorrect endpoint or region.
- Invalid bucket configuration.
- Missing permissions.
- Unsupported request parameters.

**Solution:**

Inspect backend logs, verify environment variables, and confirm that the storage client is configured with the correct endpoint and credentials.

Also verify compatibility of ACL settings with the destination storage configuration.

---

## 10. Best Practices for a Successful Migration

A cloud storage migration should be treated as an infrastructure change rather than simply a file-copying operation.

The following practices help reduce operational risks.

**1. Always maintain backups**

Keep a recoverable copy of the original storage and database before migration.

**2. Verify data integrity**

Do not assume that a successful copy command guarantees complete migration. Compare object counts, sizes, and checksums.

**3. Centralize storage configuration**

Use a dedicated storage service instead of scattering provider-specific URLs and credentials throughout the application.

**4. Separate public and private assets**

Not every uploaded file should be publicly accessible. Apply access policies according to the sensitivity of the data.

**5. Use environment-based configuration**

Keep credentials, endpoints, regions, and bucket names outside application source code.

**6. Test before production deployment**

Perform the migration in a staging environment before modifying production infrastructure.

**7. Plan for rollback**

Maintain the original storage environment until the new provider has been verified.

**8. Monitor application performance**

Observe upload latency, download response times, CDN cache behavior, error rates, and storage costs after migration.


*Key takeaway: A successful cloud storage migration is not just about moving files. It is about ensuring that every application component—from the backend upload service to the frontend image renderer and database references—continues working reliably with the new infrastructure.*
