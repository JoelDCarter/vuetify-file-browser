<template>
    <v-toolbar flat dense color="blue-grey lighten-5">
        <v-toolbar-items>
            <v-menu offset-y v-if="storages.length > 1">
                <template v-slot:activator="{ on }">
                    <v-btn icon class="storage-select-button mr-3" v-on="on">
                        <v-icon>mdi-arrow-down-drop-circle-outline</v-icon>
                    </v-btn>
                </template>
                <v-list class="storage-select-list">
                    <v-list-item
                        v-for="(item, index) in storages"
                        :key="index"
                        :disabled="item.code === storageObject.code"
                        @click="changeStorage(item.code)"
                    >
                        <v-list-item-icon>
                            <v-icon v-text="item.icon"></v-icon>
                        </v-list-item-icon>
                        <v-list-item-title>{{ item.name }}</v-list-item-title>
                    </v-list-item>
                </v-list>
            </v-menu>
            <v-btn text :input-value="path === '/'" @click="changePath('/')">
                <v-icon class="mr-2">{{storageObject.icon}}</v-icon>
                {{root.name || storageObject.name}}
            </v-btn>
            <template v-for="(segment, index) in pathSegments">
                <v-icon :key="index + '-icon'">mdi-chevron-right</v-icon>
                <v-btn
                    text
                    :input-value="index === pathSegments.length - 1"
                    :key="index + '-btn'"
                    @click="changePath(segment.path)"
                >{{ segment.name }}</v-btn>
            </template>
        </v-toolbar-items>
        <div class="flex-grow-1"></div>

        <template v-if="$vuetify.breakpoint.smAndUp">
            <v-tooltip bottom v-if="pathSegments.length > 0">
                <template v-slot:activator="{ on }">
                    <v-btn icon @click="goUp" v-on="on">
                        <v-icon>mdi-arrow-up-bold-outline</v-icon>
                    </v-btn>
                </template>
                <span v-if="pathSegments.length === 1">Up to "{{root.name}}"</span>
                <span v-else>Up to "{{pathSegments[pathSegments.length - 2].name}}"</span>
            </v-tooltip>
            <v-menu
                v-model="newFolderPopper"
                :close-on-content-click="false"
                :nudge-width="200"
                offset-y
            >
                <template v-slot:activator="{ on }">
                    <v-btn v-if="path" icon v-on="on" title="Create Folder">
                        <v-icon>mdi-folder-plus-outline</v-icon>
                    </v-btn>
                </template>
                <v-card>
                    <v-card-text>
                        <v-text-field label="Name" v-model="newFolderName" hide-details></v-text-field>
                    </v-card-text>
                    <v-card-actions>
                        <div class="flex-grow-1"></div>
                        <v-btn @click="newFolderPopper = false" depressed>Cancel</v-btn>
                        <v-btn
                            color="success"
                            :disabled="!newFolderName"
                            depressed
                            @click="mkdir"
                        >Create Folder</v-btn>
                    </v-card-actions>
                </v-card>
            </v-menu>
            <v-btn v-if="path" icon @click="$refs.inputUpload.click()" title="Upload Files">
                <v-icon>mdi-plus-circle</v-icon>
                <input v-show="false" ref="inputUpload" type="file" multiple @change="addFiles" />
            </v-btn>
        </template>
    </v-toolbar>
</template>

<script>
export default {
    props: {
        storages: Array,
        storage: String,
        root: Object,
        path: String,
        endpoints: Object,
        axios: Function
    },
    data() {
        return {
            newFolderPopper: false,
            newFolderName: ""
        };
    },
    computed: {
        pathSegments() {
            let path = this.root.path + "/",
                isFolder = this.path[this.path.length - 1] === "/",
                pathToSegment = this.path.startsWith(this.root.path) ? this.path.slice(this.root.path.length) : this.path,
                segments = pathToSegment.split("/").filter(item => item);

            segments = segments.splice(0, isFolder ? segments.length : segments.length - 1).map((item, index) => {
                path +=
                    item + (index < segments.length - 1 || isFolder ? "/" : "");
                return {
                    name: item,
                    path
                };
            });

            return segments;
        },
        storageObject() {
            return this.storages.find(item => item.code == this.storage);
        }
    },
    methods: {
        changeStorage(code) {
            if (this.storage != code) {
                this.$emit("storage-changed", code);
                this.$emit("path-changed", "");
            }
        },
        changePath(path) {
            this.$emit("path-changed", path);
        },
        goUp() {
            let segments = this.pathSegments,
                path =
                    segments.length === 1
                        ? this.root.path + "/"
                        : segments[segments.length - 2].path;
            this.changePath(path);
        },
        async addFiles(event) {
            this.$emit("add-files", event.target.files);
            this.$refs.inputUpload.value = "";
        },
        async mkdir() {
            this.$emit("loading", true);
            let url = this.endpoints.mkdir.url
                .replace(new RegExp("{storage}", "g"), this.storage)
                .replace(new RegExp("{path}", "g"), this.path + this.newFolderName);

            let config = {
                url,
                method: this.endpoints.mkdir.method || "post"
            };

            await this.axios.request(config);
            this.$emit("folder-created", this.newFolderName);
            this.newFolderPopper = false;
            this.newFolderName = "";
            this.$emit("loading", false);
        }
    }
};
</script>

<style lang="scss" scoped>
/* Storage Menu - alternate appearance
.storage-select-button ::v-deep .v-btn__content {
    flex-wrap: wrap;
    font-size: 11px;

    .v-icon {
        width: 100%;
        font-size: 22px;
    }
}
*/

.storage-select-list .v-list-item--disabled {
    background-color: rgba(0, 0, 0, 0.25);
    color: #fff;

    .v-icon {
        color: #fff;
    }
}
</style>