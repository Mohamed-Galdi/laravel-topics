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

## Other useful plugins, such as `filepond-plugin-image-preview`, can be found in the [FilePond documentation](https://pqina.nl/filepond/docs/api/plugins/).

## Step 2: Create a File Upload Component

We'll create a reusable Vue component to handle file uploads. This component will support multiple files, file validation, localization, and integration with Laravel's backend.

### `FileUpload.vue`

```js
<script setup>
import { ref, onMounted } from "vue";
import vueFilePond from "vue-filepond";
import FilePondPluginFileValidateType from "filepond-plugin-file-validate-type";
import FilePondPluginFileValidateSize from 'filepond-plugin-file-validate-size';
import "filepond/dist/filepond.min.css";
import { router, usePage } from "@inertiajs/vue3";
import { setOptions } from "vue-filepond";
import fr_FR from "filepond/locale/fr-fr";

// Define component props
const props = defineProps({
    initialFile: {
        type: [String, Object],
        default: null,
    },
    allowedFileTypes: {
        type: Array,
        default: () => ["application/pdf"],
    },
    allowMultiple: {
        type: Boolean,
        default: false,
    },
    maxFiles: {
        type: Number,
        default: 1,
    },
    locale: {
        type: Object,
        default: () => fr_FR,
    },
});

// Reactive variables
const $page = usePage();
const files = ref([]);
const FilePond = vueFilePond(
    FilePondPluginFileValidateType,
    filepond-plugin-file-validate-size
);
setOptions(props.locale);

// Handle initial file population (for update operations)
onMounted(() => {
    if (props.initialFile) {
        files.value = [{
            source: props.initialFile,
            options: { type: "local" },
        }];
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

Now that we have created our `FileUpload` component, let's see how to use it in a parent component like `Index.vue`. This section will demonstrate how to integrate the component for both creating and editing posts.

```html
<script setup>
  import { useForm } from "@inertiajs/vue3";
  import { ref } from "vue";
  import FileUpload from "@/Components/FileUpload.vue";
  import es_ES from "filepond/locale/es-ES";

  // ################################################# Create Post
  const tempFile = ref(null);
  function handleFileUploaded(fileFolder) {
    tempFile.value = fileFolder;
  }

  function handleFileReverted() {
    tempFile.value = null;
  }

  const createForm = useForm({
    title: "",
    description: "",
    postFileId: "",
  });

  function handleCreatePost() {
    createForm.postFileId = tempFile.value;
    createForm.post(route("posts.store"));
  }

  // ################################################# Edit Post
  const editForm = useForm({
    id: null,
    title: "",
    description: "",
    postFileId: "",
    postImage: "",
  });

  function openEditModal(post) {
    editForm.reset();
    editForm.id = post.id;
    editForm.title = post.title;
    editForm.description = post.description;
    editForm.postImage = post.post_image
      ? `${appUrl}storage/${post.post_image}`
      : "";
  }

  function handleEditPost() {
    editForm.postFileId = tempFile.value;
    editForm.post(route("posts.update", { id: editForm.id }));
  }
</script>

<template>
  <div>
    <!-- Create Post -->
    <FileUpload
      :locale="es_ES"
      :allowed-file-types="['application/pdf']"
      :allow-multiple="true"
      :max-files="3"
      @file-uploaded="handleFileUploaded"
      @file-reverted="handleFileReverted"
    />

    <!-- Edit Post -->
    <FileUpload
      :locale="es_ES"
      :allowed-file-types="['application/pdf']"
      :allow-multiple="true"
      :max-files="3"
      :initial-file="editForm.postImage"
      @file-uploaded="handleFileUploaded"
      @file-reverted="handleFileReverted"
    />
  </div>
</template>
```

## Step 5: Handling File Upload in the Controller

In the `PostController`, we handle the uploaded file by moving it from the temporary storage to the permanent storage location. Here’s how we do it:

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;
use App\Models\Post;
use App\Models\TempFile;

class PostController extends Controller
{
    public function store(Request $request)
    {
        $request->validate([
            'title' => 'required|unique:posts,title',
            'postFileId' => 'required',
        ]);

        $post = Post::create([ 'title' => $request->title ]);

        $this->moveFileFromTemp($post, $request->postFileId);

        return response()->json(['message' => 'Post created successfully']);
    }

    public function update(Request $request, $id)
    {
        $request->validate(['title' => 'required|unique:posts,title,' . $id]);

        $post = Post::findOrFail($id);
        $post->title = $request->title;

        if ($request->postFileId) {
            $this->deleteOldFile($post);
            $this->moveFileFromTemp($post, $request->postFileId);
        } elseif ($request->postImage === null) {
            $this->deleteOldFile($post);
        }

        $post->save();

        return response()->json(['message' => 'Post updated successfully']);
    }

    private function moveFileFromTemp(Post $post, $folder)
    {
        $tempFile = TempFile::where('folder', $folder)->first();
        if (!$tempFile) return;

        $slug = Str::slug($post->title);
        $postPath = "posts/{$post->id}-{$slug}/";

        Storage::disk('public')->move($tempFile->path, $postPath . $tempFile->name);
        $post->pdf_file = $postPath . $tempFile->name;
        $post->save();

        Storage::disk('public')->deleteDirectory('TempFiles/' . $tempFile->folder);
        $tempFile->delete();
    }

    private function deleteOldFile(Post $post)
    {
        if ($post->pdf_file) {
            $oldDirectory = dirname($post->pdf_file);
            if (Storage::disk('public')->exists($oldDirectory)) {
                Storage::disk('public')->deleteDirectory($oldDirectory);
            }
            $post->pdf_file = null;
        }
    }
}
```

This ensures a smooth file upload process while keeping storage clean and organized.
