<template>
    <PermissionList
        ref="permissionList"
        :permission-entries="permissionEntries"
        :item-manager="itemManager"
        node-name="global"
        :remove-permission="deletePermission"
        :update-permission="updatePermission"
        :create-permission="createPermission"
    >
    </PermissionList>
</template>
<script lang="ts" setup>
import { ItemManager } from "@/util/itemManager";
import PermissionList, {
    CreatePermissionFunctionInput,
    UpdatePermissionFunctionInput
} from "@/components/PermissionList.vue";
import { useClient } from "@/graphql/client";
import {
    PermissionEntry,
    GlobalPermissionOrderField,
    DefaultGlobalPermissionInfoFragment,
    GlobalPermissionOrder
} from "@/graphql/generated";
import { IdObject } from "@/util/types";
import { computed, useTemplateRef } from "vue";
import { useRoute } from "vue-router";

const client = useClient();
const route = useRoute();

const globalId = computed(() => route.params.trackable as string);
const permissionList = useTemplateRef("permissionList");

const permissionEntries = Object.values(PermissionEntry);

class PermissionItemManager extends ItemManager<DefaultGlobalPermissionInfoFragment, GlobalPermissionOrderField> {
    protected async fetchItems(
        filter: string | undefined,
        orderBy: GlobalPermissionOrder[],
        count: number,
        page: number
    ): Promise<[DefaultGlobalPermissionInfoFragment[], number]> {
        if (filter == undefined) {
            const res = await client.getGlobalPermissionList({
                orderBy,
                count,
                skip: page * count,
                filter: permissionList.value?.userFilter
            });
            const permissions = res.globalPermissions;
            return [permissions.nodes, permissions.totalCount];
        } else {
            const res = await client.getFilteredGlobalPermissionList({
                query: filter,
                count,
                filter: permissionList.value?.userFilter
            });
            return [res.searchGlobalPermissions, res.searchGlobalPermissions.length];
        }
    }
}

const itemManager = new PermissionItemManager() as ItemManager<
    DefaultGlobalPermissionInfoFragment,
    GlobalPermissionOrderField
>;

async function deletePermission(id: string): Promise<void> {
    await client.deleteGlobalPermission({ globalPermission: id });
}

async function updatePermission(input: UpdatePermissionFunctionInput<PermissionEntry>): Promise<void> {
    await client.updateGlobalPermission({ input });
}

async function createPermission(input: CreatePermissionFunctionInput<PermissionEntry>): Promise<IdObject> {
    const res = await client.createGlobalPermission({
        input: { ...input }
    });
    return res.createGlobalPermission.globalPermission;
}
</script>
