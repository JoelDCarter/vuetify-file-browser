<template>
    <v-card class="mx-auto" :loading="loading > 0">
        <confirm ref="confirm"></confirm>
        <toolbar
            :root="root"
            :path="path"
            :storages="storagesArray"
            :storage="activeStorage"
            :endpoints="endpoints"
            :axios="axiosInstance"
            :showFolderUpInToolbar="showFolderUpInToolbar"
            :readOnly="readOnlyFlags"
            :renamePending="renamePending"
            :validNameRegex="validNameRegex"
            v-on:storage-changed="storageChanged"
            v-on:path-changed="pathChanged"
            v-on:add-files="addUploadingFiles"
            v-on:folder-created="refreshPending = true"
        ></toolbar>
        <v-row no-gutters>
            <v-col v-if="tree && $vuetify.breakpoint.smAndUp" sm="auto" md="4">
                <tree
                    ref="tree"
                    :root="root"
                    :path="path"
                    :storage="activeStorage"
                    :icons="icons"
                    :itemKey="itemKey"
                    :endpoints="endpoints"
                    :axios="axiosInstance"
                    :enableFolderRefresh="enableRefreshInTree"
                    :refreshPending="refreshPending"
                    :renamePending="renamePending"
                    :showFiles="showFilesInTree"
                    :showRoot="showRootInTree"
                    v-on:path-changed="pathChanged"
                    v-on:loading="loadingChanged"
                    v-on:refreshed="refreshPending = false"
                ></tree>
            </v-col>
            <v-divider v-if="tree" vertical></v-divider>
            <v-col>
                <list
                    :root="root"
                    :path="path"
                    :storage="activeStorage"
                    :icons="icons"
                    :itemKey="itemKey"
                    :endpoints="endpoints"
                    :axios="axiosInstance"
                    :refreshPending="refreshPending"
                    :renamePending="renamePending"
                    :readOnly="readOnlyFlags"
                    :validNameRegex="validNameRegex"
                    v-on:path-changed="pathChanged"
                    v-on:loading="loadingChanged"
                    v-on:refreshed="refreshPending = false"
                    v-on:file-selected="(item) => $emit('file-selected', item)"
                    v-on:file-opened="(item) => $emit('file-opened', item)"
                    v-on:item-deleting="(item, setMessage) => $emit('item-deleting', item, setMessage)"
                    v-on:item-deleted="refreshPending = true"
                    v-on:item-moved="renamePending = false; refreshPending = true"
                    v-on:item-renaming="itemRenaming"
                    v-on:item-renamed="renamePending = false"
                    v-on:folder-refreshing="folderRefreshing"
                ></list>
            </v-col>
        </v-row>
        <upload
            v-if="uploadingFiles !== false"
            :root="root"
            :path="path"
            :storages="storagesArray"
            :storage="activeStorage"
            :files="uploadingFiles"
            :icons="icons"
            :axios="axiosInstance"
            :endpoint="endpoints.upload"
            :maxUploadFilesCount="maxUploadFilesCount"
            :maxUploadFileSize="maxUploadFileSize"
            v-on:add-files="addUploadingFiles"
            v-on:remove-file="removeUploadingFile"
            v-on:clear-files="uploadingFiles = []"
            v-on:cancel="uploadingFiles = false"
            v-on:uploaded="uploaded"
        ></upload>
    </v-card>
</template>

<script>
import axios from "axios";

import Toolbar from "./Toolbar.vue";
import Tree from "./Tree.vue";
import List from "./List.vue";
import Upload from "./Upload.vue";
import Confirm from "./Confirm.vue";

const availableStorages = [
    {
        name: "Local",
        code: "local",
        icon: "mdi-folder-multiple-outline"
    },
    {
        name: "Amazon S3",
        code: "s3",
        icon: "mdi-amazon-drive"
    }
    /*{
        name: "Dropbox",
        code: "dropbox",
        icon: "mdi-dropbox"
    }*/
];

const endpoints = {
    list: { url: "/storage/{storage}/list?path={path}", method: "get" },
    upload: { url: "/storage/{storage}/upload?path={path}", method: "post" },
    mkdir: { url: "/storage/{storage}/mkdir?path={path}", method: "post" },
    delete: { url: "/storage/{storage}/delete?path={path}", method: "post" },
    move: { url: "/storage/{storage}/move", method: "post" }
};

const fileIcons = {
    zip: "mdi-folder-zip-outline",
    rar: "mdi-folder-zip-outline",
    htm: "mdi-language-html5",
    html: "mdi-language-html5",
    js: "mdi-nodejs",
    json: "mdi-json",
    md: "mdi-markdown",
    pdf: "mdi-file-pdf",
    png: "mdi-file-image",
    jpg: "mdi-file-image",
    jpeg: "mdi-file-image",
    mp4: "mdi-filmstrip",
    mkv: "mdi-filmstrip",
    avi: "mdi-filmstrip",
    wmv: "mdi-filmstrip",
    mov: "mdi-filmstrip",
    txt: "mdi-file-document-outline",
    xls: "mdi-file-excel",
    other: "mdi-file-outline"
};

const root = { 
    path: "", 
    name: null 
};

export default {
    components: {
        Toolbar,
        Tree,
        List,
        Upload,
        Confirm
    },
    model: {
        prop: "path",
        event: "change"
    },
    props: {
        // comma-separated list of active storage codes
        storages: {
            type: String,
            default: () => availableStorages.map(item => item.code).join(",")
        },
        // code of default storage
        storage: { type: String, default: "local" },
        // show tree view
        tree: { type: Boolean, default: true },
        // file icons set
        icons: { type: Object, default: () => fileIcons },
        // item key to use for comparison
        itemKey: { type: String, default: "path" },
        // custom backend endpoints
        endpoints: { type: Object, default: () => endpoints },
        // custom axios instance
        axios: { type: Function },
        // custom configuration for internal axios instance
        axiosConfig: { type: Object, default: () => {} },
        // max files count to upload at once. Unlimited by default
        maxUploadFilesCount: { type: Number, default: 0 },
        // max file size to upload. Unlimited by default
        maxUploadFileSize: { type: Number, default: 0 },
        // root node
        root: { type: Object, default: () => root },
        // starting path
        initialPath: { type: String, default: "/" },
        // indicate whether files should be displayed in the tree
        enableRefreshInTree: {type: Boolean, default: false },
        // indicate whether files should be displayed in the tree
        showFilesInTree: {type: Boolean, default: false },
        // indicate whether the root node should be displayed in the tree
        showRootInTree: {type: Boolean, default: false },
        // indicate whether the button for navigating up one level should be displayed in the toolbar
        showFolderUpInToolbar: {type: Boolean, default: false },
        // disable any functionality beyond browsing, selecting, and opening content
        readOnly: { type: [Object, Boolean], default: true },
        // valid file/folder name regular expression
        validNameRegex: { type: RegExp, default: () => /^(?!(?:COM[0-9]|CON|LPT[0-9]|NUL|PRN|AUX|com[0-9]|con|lpt[0-9]|nul|prn|aux)(\.|\z)|\s|[\.]{2,})[^\\\/:*"?<>|]+(?<![\s\.])$/ }
    },
    data() {
        return {
            loading: 0,
            path: this.initialPath,
            activeStorage: null,
            uploadingFiles: false, // or an Array of files
            refreshPending: false,
            axiosInstance: null,
            renamePending: false
        };
    },
    computed: {
        readOnlyFlags() {
            const isObject = typeof this.readOnly === 'object' && this.readOnly !== null;

            if (isObject) {
                return {
                    files: this.readOnly.hasOwnProperty("files") && this.readOnly.files,
                    folders: this.readOnly.hasOwnProperty("folders") && this.readOnly.folders,                    
                };
            } else if (typeof(this.readOnly) === "boolean") {
                return {
                    files: this.readOnly,
                    folders: this.readOnly
                };
            }

            return {
                    files: true,
                    folders: true
                };
        },
        storagesArray() {
            let storageCodes = this.storages.split(","),
                result = [];
            storageCodes.forEach(code => {
                result.push(availableStorages.find(item => item.code == code));
            });
            return result;
        }
    },
    methods: {
        async folderRefreshing(path) {
            await this.$refs.tree.refreshFolder(path);
        },
        itemRenaming(item, pending) {
            this.renamePending = pending;
            this.$emit('item-renaming', item, pending);
        },
        loadingChanged(loading) {
            if (loading) {
                this.loading++;
            } else if (this.loading > 0) {
                this.loading--;
            }
        },
        storageChanged(storage) {
            this.activeStorage = storage;
        },
        addUploadingFiles(files) {
            files = Array.from(files);

            if (this.maxUploadFileSize) {
                files = files.filter(
                    file => file.size <= this.maxUploadFileSize
                );
            }

            if (this.uploadingFiles === false) {
                this.uploadingFiles = [];
            }
            
            if (this.maxUploadFilesCount && this.uploadingFiles.length + files.length > this.maxUploadFilesCount) {
                files = files.slice(0, this.maxUploadFilesCount - this.uploadingFiles.length);
            }

            this.uploadingFiles.push(...files);
        },
        removeUploadingFile(index) {
            this.uploadingFiles.splice(index, 1);
        },
        uploaded() {
            this.uploadingFiles = false;
            this.refreshPending = true;
        },
        pathChanged(path) {
            this.path = path;
            this.$emit("change", path);
        },
        async responseErrorHandler(error) {
            this.loadingChanged(false);
            if (this.renamePending) {
                this.renamePending = false;
            }
            let confirmed = await this.$refs.confirm.open(
                    "Error",
                    `The server returned the following error:<br>${(error.response.data && error.response.data.detail) || error.message}`,
                    { showCancel: false, yesText: "OK" }
                );
            return Promise.reject(error);
        }
    },
    created() {
        this.activeStorage = this.storage;
        this.axiosInstance = this.axios || axios.create(this.axiosConfig);
        this.axiosInstance.interceptors.response.use(undefined, this.responseErrorHandler);
    },
    mounted() {
        if (!this.path && !(this.tree && this.$vuetify.breakpoint.smAndUp)) {
            this.pathChanged(this.root.path + this.path);
        }
    },
    beforeDestroy() {
        this.axiosInstance.interceptors.response.eject(this.responseErrorHandler);
    }
};
</script>

<style lang="scss" scoped>
</style>