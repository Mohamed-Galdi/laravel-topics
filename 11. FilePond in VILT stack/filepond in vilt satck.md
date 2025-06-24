# File Upload with FilePond in a Laravel, Vue, and Inertia Project

- [Introduction](#introduction)
- [Step 1: Installing FilePond](#step-1-install-filepond-and-required-plugins)
- [Step 2: Create a File Upload Component](#step-2-create-a-file-upload-component)
- [Step 3: Handle File Uploads in Laravel (TempFile)](#step-3-handle-file-uploads-in-laravel-tempfile)
  - [migration](#migration)
  - [model](#model)
  - [controller](#controller)
  - [routes](#routes)
- [Step 4: Using the FileUpload Component in a Parent Component](#step-4-using-the-fileupload-component-in-a-parent-component)
- [Step 5: Handling File Upload in the Controller](#step-5-handling-file-upload-in-the-controller)
  - [a- Directly](#a-directly)
  - [b- Making the File Handling Reusable](#b-making-the-file-handling-reusable)

## Introduction

File upload is a common feature in web applications, and FilePond provides a powerful, flexible, and user-friendly solution. In this tutorial, we'll integrate FilePond into a Laravel + Vue + Inertia project. We'll cover:

- Installing FilePond and necessary plugins
- Setting up a reusable Vue component
- Handling file uploads in Laravel
- Implementing file revert functionality
- Customizing locale settings
- Populating files for update operations

This guide assumes you have a ready VILT (Vue, Inertia, Laravel, Tailwind) project.

---

## Step 1: Install FilePond and Required Plugins

First, install the necessary packages using npm:

```shell
npm install filepond vue-filepond
```

```shell
npm install filepond-plugin-file-validate-type
```

```shell
npm install filepond-plugin-file-validate-size
```

Other useful plugins, such as `filepond-plugin-image-preview`, can be found in the [FilePond documentation](https://pqina.nl/filepond/docs/api/plugins/).

## Step 2: Create a File Upload Component

We'll create a reusable Vue component to handle file uploads. This component will support multiple files, file validation, localization, and integration with Laravel's backend.

### `FileUpload.vue`

```js
<script setup>
// move those css imports to app.css or app.js
import 'filepond-plugin-image-preview/dist/filepond-plugin-image-preview.css';
import "filepond/dist/filepond.min.css";

import { ref, onMounted } from "vue";
import vueFilePond from "vue-filepond";
import FilePondPluginFileValidateType from "filepond-plugin-file-validate-type";
import FilePondPluginFileValidateSize from 'filepond-plugin-file-validate-size';
import FilePondPluginImagePreview from 'filepond-plugin-image-preview';
import { router, usePage } from "@inertiajs/vue3";
import { setOptions } from "vue-filepond";
// import ar_AR from "filepond/locale/ar-ar";
// import fr_FR from "filepond/locale/fr-fr";

// Define component props
const props = defineProps({
    initialFiles: {
        type: [String, Array],
        default: null,
    },
    allowedFileTypes: {
        type: Array,
         default: () => [
            "image/jpeg",
            "image/png",
            "image/gif",
            "image/svg+xml",
            "image/webp",
            "image/avif",
        ], // add more file types as needed
    },
    allowMultiple: {
        type: Boolean,
        default: false,
    },
    maxFiles: {
        type: Number,
        default: 1,
    },
    // locale: {
    //     type: Object,
    //     default: () => fr_FR,
    // },
});

// Reactive variables
const $page = usePage();
const files = ref([]);
const FilePond = vueFilePond(
    FilePondPluginFileValidateType,
    FilePondPluginFileValidateSize,
    FilePondPluginImagePreview
);
setOptions(props.locale);

// Handle initial file population (for update operations)
onMounted(() => {
    if (props.initialFiles) {
        if (Array.isArray(props.initialFiles)) {
            // Handle array of files
            files.value = props.initialFiles.map(file => ({
                source: file,
                options: { type: "local" }
            }));
        } else {
            // Handle single file
            files.value = [{
                source: props.initialFiles,
                options: { type: "local" },
            }];
        }
    }
});

const emit = defineEmits(["fileUploaded", "fileReverted"]);

// Handle file upload response
function handleFileUpload(response) {
    files.value.push(response);
    emit("fileUploaded", response);
    return response;
}

// Handle file revert (removal)
function handleFileRevert(uniqueId, load) {
    emit("fileReverted", uniqueId);
    router.post(route("file.revert", { id: uniqueId }));
    load();
}

// FilePond server configuration
const serverOptions = {
    process: {
        url: route("file.upload"),
        method: "POST",
        onload: handleFileUpload,
    },
    revert: handleFileRevert,
    headers: {
        "X-CSRF-TOKEN": $page.props.csrf_token,
    },
    load: (source, load, error, progress, abort, headers) => {
        fetch(source).then(response => response.blob()).then(load);
    },
};
</script>

<template>
    <div>
        <FilePond
            v-model="files"
            :server="serverOptions"
            :allow-multiple="allowMultiple"
            :accepted-file-types="allowedFileTypes"
            :max-files="maxFiles"
            credits="false"
            :locale="locale"
            :files="files"
        />
    </div>
</template>
```

### Key Features

- Supports multiple file uploads
- Validates file types
- Supports custom locales (e.g., French)
- Handles file population during update operations
- Includes file revert functionality
- Hides the "Powered by PQINA" credits

---

## Step 3: Handle File Uploads in Laravel (TempFile)

Create a `TempFile` model and migration to temporarily store uploaded files.

### Migration

```php
Schema::create('temp_files', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('folder');
    $table->string('path');
    $table->string('type');
    $table->timestamps();
});
```

### Model

```php
class TempFile extends Model
{
    protected $fillable = ['name', 'path', 'folder', 'type'];
}
```

### Controller

```php
class TempFileController extends Controller
{
    public function upload(Request $request)
    {
        $request->validate(['filepond' => 'required|file']);

        if ($request->hasFile('filepond')) {
            $file = $request->file('filepond');
            $fileName = $file->getClientOriginalName();
            $folder = uniqid();
            $path = $file->storeAs("TempFiles/$folder", $fileName, "public");

            TempFile::create([
                'name' => $fileName,
                'folder' => $folder,
                'path' => $path,
                'type' => $file->getMimeType()
            ]);

            return response()->json(['folder' => $folder]);
        }

        return response()->json(['error' => 'File upload failed'], 400);
    }

    public function revert($fileFolder)
    {
        $tempFile = TempFile::where('folder', $fileFolder)->first();
        if ($tempFile) {
            Storage::disk('public')->deleteDirectory("TempFiles/{$tempFile->folder}");
            $tempFile->delete();
        }
        return response()->json(['message' => 'File reverted']);
    }
}
```

### Routes

```php
Route::post('/upload', [TempFileController::class, 'upload'])->name('file.upload');
Route::post('/revert/{id}', [TempFileController::class, 'revert'])->name('file.revert');
```

---

## Step 4: Using the FileUpload Component in a Parent Component

Now that we have created our `FileUpload` component, let's see how to use it in a parent component like `create.vue` and `update.vue` pages.

### create.vue

```html
<script setup>
  import { useForm } from "@inertiajs/vue3";
  import { ref } from "vue";
  import FileUpload from "@/Components/FileUpload.vue";

  // ############################################## File upload
  const tempFile = ref([]);

  function handleFileUploaded(fileFolder) {
    tempFile.value.push(fileFolder);
  }

  function handleFileReverted(uniqueId) {
    tempFile.value = tempFile.value.filter((filePath) => {
      return !filePath.includes(uniqueId);
    });
  }
  // ################################################# Create Post

  const createForm = useForm({
    title: "",
    description: "",
    images: [],
  });

  function handleCreatePost() {
    createForm.images = tempFile.value;
    createForm.post(route("posts.store"));
  }
</script>

<template>
  <div>
    <!-- Create Post -->
    <FileUpload
      :allow-multiple="true"
      :max-files="3"
      @file-uploaded="handleFileUploaded"
      @file-reverted="handleFileReverted"
    />
  </div>
</template>
```

### update.vue

```html
<script setup>
  import { useForm } from "@inertiajs/vue3";
  import { ref } from "vue";
  import FileUpload from "@/Components/FileUpload.vue";

  const props = defineProps({
    post: {
      type: Object,
      required: true,
    },
  });

  const post = ref(props.post);

  // ################################################# Update Post
  const postImages = post.value.images.map((image) => {
    return "/" + image.path;
  });

  const updateForm = useForm({
    title: post.value.title,
    description: post.value.description,
    images: postImages, // Existing images
    newImages: [], // New images added during update
    removedImages: [],
  });

  function handleUpdatePost() {
    updateForm.patch(route("posts.update", post.value.id));
  }

  // ############################################## File upload
  // Track new and removed images
  const tempFile = ref([...postImages]); // Initialize with existing images

  function handleFileUploaded(fileFolder) {
    tempFile.value.push(fileFolder);
    updateForm.newImages.push(fileFolder);
    updateForm.images = tempFile.value;
  }

  function handleFileRemoved(imagePath) {
    tempFile.value = tempFile.value.filter(
      (filePath) => filePath !== imagePath
    );
    updateForm.removedImages.push(imagePath);
    updateForm.images = tempFile.value;
  }

  function handleFileReverted(uniqueId) {
    tempFile.value = tempFile.value.filter(
      (filePath) => !filePath.includes(uniqueId)
    );
    updateForm.newImages = updateForm.newImages.filter(
      (filePath) => !filePath.includes(uniqueId)
    );
    updateForm.images = tempFile.value;
  }
</script>

<template>
  <div>
    <!-- Update Post -->
    <FileUpload
      @file-uploaded="handleFileUploaded"
      @file-reverted="handleFileReverted"
      @file-removed="handleFileRemoved"
      :initialFiles="postImages"
      :allowMultiple="true"
      :maxFiles="3"
      :maxFileSize="5 * 1024 * 1024"
    />
  </div>
</template>
```

### index.vue (handles both create and update)

```html
<script setup>
  import { useForm } from "@inertiajs/vue3";
  import { ref } from "vue";
  import FileUpload from "@/Components/FileUpload.vue";

  const props = defineProps({
    post: {
      type: Object,
      default: null, // null when creating, object when updating
    },
  });

  // ############################################## CREATE FORM
  const createTempFiles = ref([]);
  const createForm = useForm({
    title: "",
    description: "",
    images: [],
  });

  function handleCreateFileUploaded(fileFolder) {
    createTempFiles.value.push(fileFolder);
  }

  function handleCreateFileReverted(uniqueId) {
    createTempFiles.value = createTempFiles.value.filter((filePath) => {
      return !filePath.includes(uniqueId);
    });
  }

  function handleCreatePost() {
    createForm.images = createTempFiles.value;
    createForm.post(route("posts.store"));
  }

  // ############################################## UPDATE FORM
  const updateTempFiles = ref([]);
  const updateForm = ref(null);

  // Initialize update form if post exists
  if (props.post) {
    const postImages = props.post.images.map((image) => "/" + image.path);
    updateTempFiles.value = [...postImages];

    updateForm.value = useForm({
      title: props.post.title,
      description: props.post.description,
      images: postImages,
      newImages: [],
      removedImages: [],
    });
  }

  function handleUpdateFileUploaded(fileFolder) {
    updateTempFiles.value.push(fileFolder);
    updateForm.value.newImages.push(fileFolder);
    updateForm.value.images = updateTempFiles.value;
  }

  function handleUpdateFileRemoved(imagePath) {
    updateTempFiles.value = updateTempFiles.value.filter(
      (filePath) => filePath !== imagePath
    );
    updateForm.value.removedImages.push(imagePath);
    updateForm.value.images = updateTempFiles.value;
  }

  function handleUpdateFileReverted(uniqueId) {
    updateTempFiles.value = updateTempFiles.value.filter(
      (filePath) => !filePath.includes(uniqueId)
    );
    updateForm.value.newImages = updateForm.value.newImages.filter(
      (filePath) => !filePath.includes(uniqueId)
    );
    updateForm.value.images = updateTempFiles.value;
  }

  function handleUpdatePost() {
    updateForm.value.post(route("posts.update", props.post.id));
  }

</script>

<template>
  <div>
    <!-- CREATE FORM -->
    <FileUpload
      key="create-upload"
      :allow-multiple="true"
      :max-files="3"
      @file-uploaded="handleCreateFileUploaded"
      @file-reverted="handleCreateFileReverted"
    />

    <!-- UPDATE FORM -->
    <FileUpload
      key="update-upload"
      :initial-files="post?.images?.map(img => '/' + img.path) || []"
      :allow-multiple="true"
      :max-files="3"
      @file-uploaded="handleUpdateFileUploaded"
      @file-reverted="handleUpdateFileReverted"
      @file-removed="handleUpdateFileRemoved"
    />
  </div>
</template>
```

## Step 5: Handling File Upload in the Controller

### a. FileService

To avoid repeating the same file handling logic across multiple places, you can extract it into a **service class** or a **helper function**.

You can create a `FileService.php` inside `app/Services/`:

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;
use App\Models\TempFile;

class FileService
{
    public static function moveTempFile(string $folder, string $destinationPath, string $title): ?string
    {
        $tempFile = TempFile::where('folder', $folder)->first();

        if ($tempFile) {
            $slug = Str::slug($title);
            $uniqueName = "{$slug}-" . time() . "-" . Str::random(8) . "." . pathinfo($tempFile->name, PATHINFO_EXTENSION);
            $filePath = "{$destinationPath}/{$uniqueName}";

            Storage::disk('public')->move($tempFile->path, $filePath);

            Storage::disk('public')->deleteDirectory("TempFiles/{$tempFile->folder}");
            $tempFile->delete();

            $finalFilePath = '/storage/' . $filePath;

            return $finalFilePath;
        }

        return null;
    }
}
```

### b. On the controller

The `store` method can be like this:

```php
foreach ($request->images as $image) {
    $path = FileService::moveTempFile($image, "post_images/{$postImagesFolderName}", $post->id);
    $post->images()->create(['path' => $path,]);
}
```

The `update` method can be like this:

```php
// Handle removed images
    if ($request->removedImages) {
        // Fix path format to match disk structure
        $imagePaths = array_map(function ($path) {
            return preg_replace('#^/?storage/#', '', $path);
        }, $request->removedImages);
        // Bulk delete from database
        $post->images()->whereIn('path', array_map(fn ($path) => 'storage/'.$path, $imagePaths))->delete();
        // Delete from storage
        foreach ($imagePaths as $imagePath) {
            if (Storage::disk('public')->exists($imagePath)) {
                Storage::disk('public')->delete($imagePath);
                // Clean up empty folders
                $folderPath = dirname($imagePath);
                if (
                    Storage::disk('public')->exists($folderPath) &&
                    count(Storage::disk('public')->files($folderPath)) === 0
                ) {
                    Storage::disk('public')->deleteDirectory($folderPath);
                }
            } else {
                Log::warning("Image not found in storage: {$imagePath}");
            }
        }
    }
    // Handle new images
    if ($request->newImages) {
        $folderName = $this->getImageFolder($request->images, $post->id);
        foreach ($request->newImages as $image) {
            // Corrected parameter order to match FileService
            $path = FileService::moveTempFile($image, "post_images/{$folderName}", $post->id);
            if ($path) {
                $post->images()->create(['path' => $path]);
            } else {
                Log::warning("Failed to move temporary file: {$image}");
            }
        }
    }

    // another method in the controller
    private function getImageFolder(array $images, string $propertyId): string
    {
        // Extract only the non-temporary images (those that already have paths)
        $existingImages = array_filter($images, function ($image) {
            return is_string($image) && strpos($image, 'storage/property_images/') !== false;
        });

        // If we have existing images, extract the folder name from the first one
        if (! empty($existingImages)) {
            $firstImage = reset($existingImages);

            // Extract folder name using regex - matches the pattern in storage/property_images/FOLDER_NAME/filename
            if (preg_match('#property_images/([^/]+)/#', $firstImage, $matches)) {
                return $matches[1];
            }
        }

        // If no existing folder found, create a new one with a simpler unique name
        return 'property_'.$propertyId.'_'.Str::random(8);
    }
```