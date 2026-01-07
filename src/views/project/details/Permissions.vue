<template>
    <PermissionList
        ref="permissionList"
        :permission-entries="permissionEntries"
        :item-manager="itemManager"
        node-name="project"
        :remove-permission="removePermission"
        :update-permission="updatePermission"
        :create-permission="createPermission"
    >
        <template #import-dialog="{ importedPermission }">
            <ImportProjectPermissionDialog :project="projectId" @imported-project-permission="importedPermission" />
        </template>
    </PermissionList>
</template>
<script lang="ts" setup>
import PermissionList, {
    CreatePermissionFunctionInput,
    UpdatePermissionFunctionInput
} from "@/components/PermissionList.vue";
import ImportProjectPermissionDialog from "@/components/dialog/ImportProjectPermissionDialog.vue";
import { NodeReturnType, useClient } from "@/graphql/client";
import {
    ProjectPermissionEntry,
    ProjectPermissionOrderField,
    DefaultProjectPermissionInfoFragment,
    ProjectPermissionOrder
} from "@/graphql/generated";
import { IdObject } from "@/util/types";
import { computed, useTemplateRef } from "vue";
import { useRoute } from "vue-router";
import { ItemManager } from "@/util/itemManager";

const client = useClient();
const route = useRoute();

const projectId = computed(() => route.params.trackable as string);
const permissionList = useTemplateRef("permissionList");

const permissionEntries = Object.values(ProjectPermissionEntry);

class ProjectPermissionItemManager extends ItemManager<
    DefaultProjectPermissionInfoFragment,
    ProjectPermissionOrderField
> {
    protected async fetchItems(
        filter: string | undefined,
        orderBy: ProjectPermissionOrder[],
        count: number,
        page: number
    ): Promise<[DefaultProjectPermissionInfoFragment[], number]> {
        if (filter == undefined) {
            const res = await client.getProjectPermissionList({
                orderBy,
                count,
                skip: page * count,
                project: projectId.value,
                filter: permissionList.value?.userFilter
            });
            const permissions = (res.node as NodeReturnType<"getProjectPermissionList", "Project">).permissions;
            return [permissions.nodes, permissions.totalCount];
        } else {
            const res = await client.getFilteredProjectPermissionList({
                query: filter,
                count,
                filter: {
                    ...permissionList.value?.userFilter,
                    nodesWithPermission: { any: { id: { eq: projectId.value } } }
                }
            });
            return [res.searchProjectPermissions, res.searchProjectPermissions.length];
        }
    }
}
const itemManager: ItemManager<DefaultProjectPermissionInfoFragment, ProjectPermissionOrderField> =
    new ProjectPermissionItemManager();

async function removePermission(id: string): Promise<void> {
    await client.removeProjectPermissionFromProject({ project: projectId.value, projectPermission: id });
}

async function updatePermission(input: UpdatePermissionFunctionInput<ProjectPermissionEntry>): Promise<void> {
    await client.updateProjectPermission({ input });
}

async function createPermission(input: CreatePermissionFunctionInput<ProjectPermissionEntry>): Promise<IdObject> {
    const res = await client.createProjectPermission({
        input: { nodesWithPermission: [projectId.value], ...input }
    });
    return res.createProjectPermission.projectPermission;
}
</script>
