<template>
    <PermissionList
        ref="permissionList"
        :permission-entries="permissionEntries"
        :item-manager="itemManager"
        node-name="component"
        :remove-permission="removePermission"
        :update-permission="updatePermission"
        :create-permission="createPermission"
    >
        <template #import-dialog="{ importedPermission }">
            <ImportComponentPermissionDialog
                :component="componentId"
                @imported-component-permission="importedPermission"
            />
        </template>
    </PermissionList>
</template>
<script lang="ts" setup>
import PermissionList, {
    CreatePermissionFunctionInput,
    UpdatePermissionFunctionInput
} from "@/components/PermissionList.vue";
import ImportComponentPermissionDialog from "@/components/dialog/ImportComponentPermissionDialog.vue";
import { NodeReturnType, useClient } from "@/graphql/client";
import {
    ComponentPermissionEntry,
    ComponentPermissionOrder,
    ComponentPermissionOrderField,
    DefaultComponentPermissionInfoFragment
} from "@/graphql/generated";
import { ItemManager } from "@/util/itemManager";
import { IdObject } from "@/util/types";
import { computed, useTemplateRef } from "vue";
import { useRoute } from "vue-router";

const client = useClient();
const route = useRoute();

const componentId = computed(() => route.params.trackable as string);
const permissionList = useTemplateRef("permissionList");

const permissionEntries = Object.values(ComponentPermissionEntry);

class ComponentPermissionItemManager extends ItemManager<
    DefaultComponentPermissionInfoFragment,
    ComponentPermissionOrderField
> {
    protected async fetchItems(
        filter: string | undefined,
        orderBy: ComponentPermissionOrder[],
        count: number,
        page: number
    ): Promise<[DefaultComponentPermissionInfoFragment[], number]> {
        if (filter == undefined) {
            const res = await client.getComponentPermissionList({
                orderBy,
                count,
                skip: page * count,
                component: componentId.value,
                filter: permissionList.value?.userFilter
            });
            const permissions = (res.node as NodeReturnType<"getComponentPermissionList", "Component">).permissions;
            return [permissions.nodes, permissions.totalCount];
        } else {
            const res = await client.getFilteredComponentPermissionList({
                query: filter,
                count,
                filter: {
                    ...permissionList.value?.userFilter,
                    nodesWithPermission: { any: { id: { eq: componentId.value } } }
                }
            });
            return [res.searchComponentPermissions, res.searchComponentPermissions.length];
        }
    }
}

const itemManager = new ComponentPermissionItemManager() as ItemManager<
    DefaultComponentPermissionInfoFragment,
    ComponentPermissionOrderField
>;

async function removePermission(id: string): Promise<void> {
    await client.removeComponentPermissionFromComponent({ component: componentId.value, componentPermission: id });
}

async function updatePermission(input: UpdatePermissionFunctionInput<ComponentPermissionEntry>): Promise<void> {
    await client.updateComponentPermission({ input });
}

async function createPermission(input: CreatePermissionFunctionInput<ComponentPermissionEntry>): Promise<IdObject> {
    const res = await client.createComponentPermission({
        input: { nodesWithPermission: [componentId.value], ...input }
    });
    return res.createComponentPermission.componentPermission;
}
</script>
