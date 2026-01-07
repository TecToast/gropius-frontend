<template>
    <PermissionList
        ref="permissionList"
        :permission-entries="permissionEntries"
        :item-manager="itemManager"
        node-name="ims"
        :remove-permission="removePermission"
        :update-permission="updatePermission"
        :create-permission="createPermission"
    >
        <template #import-dialog="{ importedPermission }">
            <ImportIMSPermissionDialog :ims="imsId" @imported-ims-permission="importedPermission" />
        </template>
    </PermissionList>
</template>
<script lang="ts" setup>
import PermissionList, {
    CreatePermissionFunctionInput,
    UpdatePermissionFunctionInput
} from "@/components/PermissionList.vue";
import ImportIMSPermissionDialog from "@/components/dialog/ImportIMSPermissionDialog.vue";
import { NodeReturnType, useClient } from "@/graphql/client";
import {
    ImsPermissionEntry,
    ImsPermissionOrder,
    ImsPermissionOrderField,
    DefaultImsPermissionInfoFragment
} from "@/graphql/generated";
import { ItemManager } from "@/util/itemManager";
import { IdObject } from "@/util/types";
import { computed, useTemplateRef } from "vue";
import { useRoute } from "vue-router";

const client = useClient();
const route = useRoute();

const imsId = computed(() => route.params.ims as string);
const permissionList = useTemplateRef("permissionList");

const permissionEntries = Object.values(ImsPermissionEntry);

class ImsPermissionItemManager extends ItemManager<DefaultImsPermissionInfoFragment, ImsPermissionOrderField> {
    protected async fetchItems(
        filter: string | undefined,
        orderBy: ImsPermissionOrder[],
        count: number,
        page: number
    ): Promise<[DefaultImsPermissionInfoFragment[], number]> {
        if (filter == undefined) {
            const res = await client.getIMSPermissionList({
                orderBy,
                count,
                skip: page * count,
                ims: imsId.value,
                filter: permissionList.value?.userFilter
            });
            const permissions = (res.node as NodeReturnType<"getIMSPermissionList", "IMS">).permissions;
            return [permissions.nodes, permissions.totalCount];
        } else {
            const res = await client.getFilteredIMSPermissionList({
                query: filter,
                count,
                filter: {
                    ...permissionList.value?.userFilter,
                    nodesWithPermission: { any: { id: { eq: imsId.value } } }
                }
            });
            return [res.searchIMSPermissions, res.searchIMSPermissions.length];
        }
    }
}
const itemManager = new ImsPermissionItemManager() as ItemManager<
    DefaultImsPermissionInfoFragment,
    ImsPermissionOrderField
>;

async function removePermission(id: string): Promise<void> {
    await client.removeIMSPermissionFromIMS({ ims: imsId.value, imsPermission: id });
}

async function updatePermission(input: UpdatePermissionFunctionInput<ImsPermissionEntry>): Promise<void> {
    await client.updateIMSPermission({ input });
}

async function createPermission(input: CreatePermissionFunctionInput<ImsPermissionEntry>): Promise<IdObject> {
    const res = await client.createIMSPermission({
        input: { nodesWithPermission: [imsId.value], ...input }
    });
    return res.createIMSPermission.imsPermission;
}
</script>
