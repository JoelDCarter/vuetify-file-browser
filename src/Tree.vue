<template>
    <v-card flat tile min-width="250" min-height="380" class="d-flex flex-column folders-tree-card">
        <div class="grow scroll-x">
            <v-treeview
                :open="open"
                :active="active"
                :items="items"
                :search="filter"
                :load-children="readFolder"
                v-on:update:active="activeChanged"
                item-key="path"
                item-text="basename"
                dense
                activatable
                transition
                class="folders-tree"
            >
                <template v-slot:prepend="{ item, open }">
                    <v-icon
                        v-if="item.type === 'dir'"
                    >{{ open ? 'mdi-folder-open-outline' : 'mdi-folder-outline' }}</v-icon>
                    <v-icon v-else>{{ icons[item.extension.toLowerCase()] || icons['other'] }}</v-icon>
                </template>
                <template v-slot:label="{ item }">
                    {{item.basename}}
                    <v-btn
                        icon
                        v-if="enableFolderRefresh && item.type === 'dir'"
                        @click.stop="readFolder(item)"
                        class="ml-1"
                    >
                        <v-icon class="pa-0 mdi-18px" color="grey lighten-1">mdi-refresh</v-icon>
                    </v-btn>
                </template>
            </v-treeview>
        </div>
        <v-divider></v-divider>
        <v-toolbar dense flat class="shrink">
            <v-text-field
                solo
                flat
                hide-details
                label="Filter"
                v-model="filter"
                prepend-inner-icon="mdi-filter-outline"
                class="ml-n3"
            ></v-text-field>
            <v-tooltip top>
                <template v-slot:activator="{ on }">
                    <v-btn icon @click="init" v-on="on">
                        <v-icon>mdi-collapse-all-outline</v-icon>
                    </v-btn>
                </template>
                <span>Collapse All</span>
            </v-tooltip>
        </v-toolbar>
    </v-card>
</template>

<script>
import _findKey from 'lodash.findkey';

export default {
    props: {
        icons: Object,
        itemKey: String,
        storage: String,
        root: Object,
        path: String,
        endpoints: Object,
        axios: Function,
        enableFolderRefresh: Boolean,
        refreshPending: Boolean,
        renamePending: Boolean,
        showFiles: Boolean,
        showRoot: Boolean
    },
    computed: {
        rootItemDefaultValue() {
            return { 
                type: "dir", 
                path: this.root.path + "/", 
                basename: this.root.name || "root",
                extension: "",
                name: this.root.name || "root",
                children: []
            }
        }
    },
    data() {
        return {
            open: [],
            active: [],
            items: [],
            filter: "",
            rootItem: this.rootItemDefaultValue
        };
    },
    methods: {
        init() {
            this.open = [];
            this.active = [];
            this.items = [];
            this.rootItem = this.rootItemDefaultValue;
            // set default files tree items (root item) in nextTick.
            // Otherwise this.open isn't cleared properly (due to syncing perhaps)
            setTimeout(async () => {
                let pathSegments = this.path.split("/").filter(segment => segment);
                let path = await this.recurseSubFolders(this.rootItem, pathSegments);
                if (this.showRootInTree) {
                    this.items = this.rootItem;
                } else {
                    this.mergeFolders(this.rootItem.children, this.items); 
                }
                this.$emit("path-changed", path);
            }, 0);
        },
        async recurseSubFolders(item, pathSegments) {
            await this.readFolder(item);
            if (pathSegments.length > 0) {
                if (item !== this.rootItem && !this.open.includes(item.path)) {
                    this.open.push(item.path);
                }
                let pathSegment = pathSegments.shift().toLowerCase();
                if (pathSegment) {
                    let nextItem = item.children.find(child => child.name.toLowerCase() === pathSegment && child.type === "dir")
                    if (nextItem) {
                        return await this.recurseSubFolders(nextItem, pathSegments);
                    }
                }
            }
            return item.path;
        },
        async readFolder(item) {
            this.$emit("loading", true);
            let url = this.endpoints.list.url
                .replace(new RegExp("{storage}", "g"), this.storage)
                .replace(new RegExp("{path}", "g"), item.path);

            let config = {
                url,
                method: this.endpoints.list.method || "get"
            };

            await this.axios.request(config)
                .then((response) => {
                    // eslint-disable-next-line require-atomic-updates
                    this.mergeFolders(
                        response.data
                        .filter(item => item.type === "dir" || this.showFiles), 
                        item.children);
                        
                    this.$emit("loading", false);
                });
        },
        activeChanged(active) {
            let rootPath = this.root.path ? this.root.path + "/" : "";
            let currentPath = active.length ? active[0] : this.path;
            let relativePath = currentPath.startsWith(rootPath) ? currentPath.replace(rootPath, "") : currentPath;
            let pathSegments = relativePath.split("/").filter(segment => segment);
            let relativePathSegment = rootPath;

            if (active.length === 0 && currentPath !== rootPath) {
                active.push(currentPath);
            }

            for (let i = 0; i < pathSegments.length; i++) {
                relativePathSegment += pathSegments[i] + "/";
                if (!this.open.includes(relativePathSegment)) {
                    this.open.push(relativePathSegment);
                }
            }

            this.active = active;               

            if (this.path !== currentPath) {
                this.$emit("path-changed", currentPath);
            }
        },
        findItem(path) {
            let stack = [];
            stack.push(this.rootItem);
            while (stack.length > 0) {
                let node = stack.pop();
                if (node.path == path) {
                    return node;
                } else if (node.children && node.children.length) {
                    for (let i = 0; i < node.children.length; i++) {
                        stack.push(node.children[i]);
                    }
                }
            }
            return null;
        },
        mergeFolders(src, dest) {
            // if we find an item by itemKey in dest that's not in the src, delete it
            let i = 0;
            let destIndicesToDelete = [];
            for (; i < dest.length; i++) {
                let missingItem = _findKey(src, [this.itemKey, dest[i][this.itemKey]]);

                if (!missingItem) {
                    destIndicesToDelete.push(i);
                }
            }
            for (i = 0; i < destIndicesToDelete.length; i++) {
                this.$delete(dest, destIndicesToDelete[i]);
            }
            // if we find an item in the src that's also in dest, then merge it's values. otherwise, add it to dest
            for (i = 0; i < src.length; i++) {
                let destItem = _findKey(dest, [this.itemKey, src[i][this.itemKey]]);

                if (destItem) {
                    // update dest item by merging props
                    Object.assign(destItem, src[i]);
                } else {
                    if (src[i].type === "dir") {
                        src[i].children = [];
                    }
                    dest.push(src[i]);
                }
            }
        },
        async refreshFolder(path) {
            let item = this.findItem(path);
            if (item) {
                await this.readFolder(item);
            }
        }
    },
    watch: {
        storage() {
            this.init();
        },
        path() {
            this.active = [this.path];
            if (!this.open.includes(this.path)) {
                this.open.push(this.path);
            }
        },
        async refreshPending() {
            if (this.refreshPending) {
                this.refreshFolder(this.path);
                this.$nextTick(() => this.$emit("refreshed"));
            }
        }
    },
    created() {
        this.init();
    }
};
</script>

<style lang="scss" scoped>
.folders-tree-card {
    height: 100%;

    .scroll-x {
        overflow-x: auto;
    }

    ::v-deep .folders-tree {
        width: fit-content;
        min-width: 250px;

        .v-treeview-node {
            cursor: pointer;

            &:hover {
                background-color: rgba(0, 0, 0, 0.02);
            }
        }
    }
}
</style>