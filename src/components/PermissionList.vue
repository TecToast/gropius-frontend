<template>
    <PaginatedList
        name="permissions"
        :item-manager="itemManager"
        :sort-fields="permissionSortFields"
        :to="() => undefined"
        :dependencies="[permissionDependencyCounter, userFilter]"
        query-param-prefix=""
    >
        <template #item="{ item }">
            <ListItem
                :title="item.name"
                :subtitle="item.description || 'No description provided'"
                :italic-subtitle="!item.description"
            >
                <template #append-line>
                    <div class="d-flex flex-wrap chip-container mt-1">
                        <v-chip
                            v-for="(entry, index) in item.entries"
                            :key="index"
                            color="primary"
                            size="small"
                            class="flex-shrink-0"
                        >
                            {{ enumToRegularCase(entry) }}
                        </v-chip>
                    </div>
                </template>
                <template #append>
                    <v-chip
                        v-if="item.allUsers"
                        color="error"
                        size="small"
                        class="flex-shrink-0 mr-5"
                        icon="mdi-account-multiple"
                        >All users</v-chip
                    >
                    <div class="text-medium-emphasis icon-container mr-5">
                        <v-icon icon="mdi-account" />
                        {{ item.users.totalCount }}
                    </div>
                    <IconButton @click="permissionToEdit = item" class="mr-2">
                        <v-icon icon="mdi-pencil" />
                        <v-tooltip activator="parent" location="bottom">Edit permission</v-tooltip>
                    </IconButton>
                    <IconButton @click="permissionToManageUsers = item" class="mr-2">
                        <v-icon icon="mdi-account-edit" />
                        <v-tooltip activator="parent" location="bottom">Manage users</v-tooltip>
                    </IconButton>
                    <IconButton>
                        <v-icon icon="mdi-close" />
                        <ConfirmationDialog
                            :title="`Remove permission from ${nodeName}?`"
                            :message="`Are you sure you want to remove the permission from this ${nodeName}?`"
                            confirm-text="Remove"
                            @confirm="removePermission(item.id)"
                        />
                        <v-tooltip activator="parent" location="bottom"
                            >Remove permission from {{ nodeName }}</v-tooltip
                        >
                    </IconButton>
                </template>
            </ListItem>
        </template>
        <template #additional-filter>
            <div class="d-flex ga-2 align-center h-100">
                <v-switch v-model="allUsers" label="Show only permissions for all users" :inset="false" />
                <FilterDropdown
                    v-model="users"
                    label="User"
                    :item-manager="itemManager"
                    :mapper="(item) => item.users.nodes"
                    :fetch-on-search="userFetch"
                />
            </div>
        </template>
        <CreatePermissionDialog
            :create-permission="createPermission"
            :permission-entries="permissionEntries"
            @created-permission="permissionDependencyCounter++"
        />
        <ManagePermissionUsersDialog
            v-model="permissionToManageUsers"
            :update-permission="updatePermission"
            @updated-permission="permissionDependencyCounter++"
        />
        <UpdatePermissionDialog
            v-model="permissionToEdit"
            :permission-entries="permissionEntries"
            :update-permission="updatePermission"
            @updated-permission="permissionDependencyCounter++"
        />
        <slot name="import-dialog" :imported-permission="importedPermission" />
    </PaginatedList>
</template>
<script
    lang="ts"
    setup
    generic="
        E extends string,
        T extends {
            id: string;
            name: string;
            description: string;
            entries: E[];
            allUsers: boolean;
            users: { totalCount: number; nodes: { id: string; name: string }[] };
        }
    "
>
import ListItem from "@/components/ListItem.vue";
import PaginatedList from "@/components/PaginatedList.vue";
import ConfirmationDialog from "@/components/dialog/ConfirmationDialog.vue";
import { enumToRegularCase } from "@/util/casingTransformers";
import { permissionSortFields } from "@/util/permissionSortFields";
import { withErrorMessage } from "@/util/withErrorMessage";
import { computed, PropType, watch } from "vue";
import { ref } from "vue";
import ManagePermissionUsersDialog from "./dialog/ManagePermissionUsersDialog.vue";
import { IdObject, ValueOf } from "@/util/types";
import CreatePermissionDialog from "./dialog/CreatePermissionDialog.vue";
import UpdatePermissionDialog from "./dialog/UpdatePermissionDialog.vue";
import { ItemManager } from "@/util/itemManager";
import FilterDropdown from "@/components/input/FilterDropdown.vue";
import { useFilterOption } from "@/util/useFilterOption";
import { useClient } from "@/graphql/client";
import { GlobalPermissionFilterInput } from "@/graphql/generated";

export type UpdatePermissionFunctionInput<T> = IdObject &
    Partial<{
        name: string;
        description: string;
        addedEntries: T[];
        removedEntries: T[];
        allUsers: boolean;
        addedUsers: string[];
        removedUsers: string[];
    }>;

export type UpdatePermissionFunction<T> = (input: UpdatePermissionFunctionInput<T>) => Promise<void>;

export type CreatePermissionFunctionInput<T> = {
    name: string;
    description: string;
    entries: T[];
    allUsers: boolean;
    users: string[];
};

export type CreatePermissionFunction<T> = (input: CreatePermissionFunctionInput<T>) => Promise<IdObject>;

const props = defineProps({
    nodeName: {
        type: String,
        required: true
    },
    itemManager: {
        type: Object as PropType<ItemManager<T, ValueOf<typeof permissionSortFields>>>,
        required: true
    },
    permissionEntries: {
        type: Array as PropType<E[]>,
        required: true
    },
    removePermission: {
        type: Function as PropType<(permissionId: string) => Promise<void>>,
        required: true
    },
    updatePermission: {
        type: Function as PropType<UpdatePermissionFunction<E>>,
        required: true
    },
    createPermission: {
        type: Function as PropType<CreatePermissionFunction<E>>,
        required: true
    }
});

const permissionDependencyCounter = ref<number>(0);
const permissionToEdit = ref<T | undefined>();
const permissionToManageUsers = ref<T | undefined>();

async function removePermission(permissionId: string) {
    await withErrorMessage(async () => {
        await props.removePermission(permissionId);
    }, `Error removing permission from ${props.nodeName}`);
    permissionDependencyCounter.value++;
}

function importedPermission(permission: IdObject) {
    permissionDependencyCounter.value++;
}

const allUsers = ref<boolean>(false);
const users = useFilterOption("users", true);
watch(allUsers, (newVal) => {
    if (newVal) {
        users.value = [];
    }
});
watch(users, (newVal) => {
    if (newVal.length > 0) {
        allUsers.value = false;
    }
});
const userFilter = computed(() => {
    const filter: GlobalPermissionFilterInput = {};
    if (allUsers.value) {
        filter.allUsers = { eq: true };
    }
    if (users.value.length > 0) {
        filter.users = { any: { id: { in: users.value } } };
    }
    return Object.keys(filter).length > 0 ? filter : undefined;
});
const client = useClient();
const userFetch = async (search: string) =>
    client
        .searchGropiusUsers({ query: search, count: 100 })
        .then((res) => res.searchGropiusUsers.map((u) => ({ id: u.id, name: u.username })));

defineExpose({
    userFilter
});
</script>
<style scoped lang="scss">
@use "@/styles/settings.scss";
.icon-container {
    min-width: settings.$icon-with-number-width;
}

.chip-container {
    row-gap: 0.5rem;
    column-gap: 0.25rem;
}
</style>
