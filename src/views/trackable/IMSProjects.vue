<template>
    <PaginatedList
        name="IMS projects"
        :item-manager="itemManager"
        :sort-fields="sortFields"
        :to="
            (imsProject: IMSProject) => (imsProject.ims ? imsProjectRoute(imsProject.id, imsProject.ims.id) : undefined)
        "
        query-param-prefix=""
        :dependencies="dependencyArray"
    >
        <template #item="{ item }">
            <ListItem
                :title="item.name"
                :subtitle="item.description || 'No description provided'"
                :italic-subtitle="!item.description"
            >
                <template #append>
                    <SyncSelfAllowedSwitch :target="item" />
                </template>
            </ListItem>
        </template>
        <template #additional-filter>
            <FilterDropdown
                v-model="templateIds"
                :item-manager="itemManager"
                :mapper="(item) => item.ims?.template"
                label="Template"
                :fetch-on-search="templateFetch"
            />
        </template>
        <CreateIMSProjectDialog
            :trackable="trackable"
            @created-ims-project="(imsProject) => selectIMSProject(imsProject)"
        />
    </PaginatedList>
</template>
<script lang="ts" setup>
import PaginatedList from "@/components/PaginatedList.vue";
import { NodeReturnType, useClient } from "@/graphql/client";
import { RouteLocationRaw, useRoute, useRouter } from "vue-router";
import ListItem from "@/components/ListItem.vue";
import { IdObject } from "@/util/types";
import {
    DefaultImsProjectInfoFragment,
    ImsProjectFilterInput,
    ImsProjectOrder,
    ImsProjectOrderField
} from "@/graphql/generated";
import SyncSelfAllowedSwitch from "@/components/input/SyncSelfAllowedSwitch.vue";
import { computed } from "vue";
import CreateIMSProjectDialog from "@/components/dialog/CreateIMSProjectDialog.vue";
import { ItemManager } from "@/util/itemManager";
import { useFilterOption } from "@/util/useFilterOption";
import FilterDropdown from "@/components/input/FilterDropdown.vue";

type IMSProject = DefaultImsProjectInfoFragment;

const client = useClient();
const router = useRouter();
const route = useRoute();

const trackable = computed(() => route.params.trackable as string);

const sortFields = {
    Name: ImsProjectOrderField.Name,
    IMS: [ImsProjectOrderField.ImsName, ImsProjectOrderField.ImsId],
    "[Default]": ImsProjectOrderField.Id
};

const templateIds = useFilterOption("template", true);
const templateInput = computed(() => {
    if (templateIds.value.length == 0) {
        return undefined;
    }
    return { id: { in: templateIds.value } };
});
const templateFetch = async (query: string) =>
    client.searchIMSTemplates({ query: query, count: 100 }).then((res) => res.searchIMSTemplates);

const dependencyArray = computed(() => [templateInput]);

class IMSProjectItemManager extends ItemManager<IMSProject, ImsProjectOrderField> {
    protected async fetchItems(
        filter: string | undefined,
        orderBy: ImsProjectOrder[],
        count: number,
        page: number
    ): Promise<[IMSProject[], number]> {
        const generalFilters: ImsProjectFilterInput = {
            ims: {
                template: templateInput.value
            }
        };
        if (filter == undefined) {
            const res = (
                await client.getIMSProjectListFromTrackable({
                    orderBy,
                    count,
                    skip: page * count,
                    trackable: trackable.value,
                    filter: generalFilters
                })
            ).node as NodeReturnType<"getIMSProjectListFromTrackable", "Component">;
            return [res.syncsTo.nodes, res.syncsTo.totalCount];
        } else {
            const res = await client.getFilteredIMSProjectList({
                query: filter,
                count,
                filter: { ...generalFilters, trackable: { id: { eq: trackable.value } } }
            });
            return [res.searchIMSProjects, res.searchIMSProjects.length];
        }
    }
}
const itemManager: ItemManager<IMSProject, ImsProjectOrderField> = new IMSProjectItemManager();

function selectIMSProject(imsProject: IdObject & { ims: IdObject }) {
    router.push(imsProjectRoute(imsProject.id, imsProject.ims.id));
}

function imsProjectRoute(id: string, ims: string): RouteLocationRaw {
    return {
        name: "ims-project-general",
        params: {
            ims,
            project: id
        }
    };
}
</script>
