<template>
    <PaginatedList
        name="versions"
        :item-manager="itemManager"
        :sort-fields="sortFields"
        :to="() => undefined"
        :dependencies="dependencyArray"
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
                        <template v-if="item.filterByTemplate.nodes.length > 0">
                            <v-chip
                                v-for="(entry, index) in item.filterByTemplate.nodes"
                                :key="index"
                                color="primary"
                                size="small"
                                class="flex-shrink-0"
                            >
                                {{ entry.name }}
                            </v-chip>
                        </template>
                        <template v-else>
                            <div class="text-medium-emphasis text-body-2 text-ellipsis font-italic flex-grow-1">
                                <v-icon icon="mdi-filter-off" />
                                No filter configured
                            </div>
                        </template>
                    </div>
                </template>
                <template #append>
                    <IconButton :disabled="!(manageViews ?? false)" @click="updateView(item)" class="mr-2">
                        <v-icon icon="mdi-pencil" />
                        <v-tooltip activator="parent" location="bottom">Edit view</v-tooltip>
                    </IconButton>
                    <IconButton :disabled="!(manageViews ?? false)">
                        <v-icon icon="mdi-delete" />
                        <ConfirmationDialog
                            :title="`Delete view?`"
                            :message="`Are you sure you want to the view?`"
                            confirm-text="Delete"
                            @confirm="deleteView(item.id)"
                        />
                        <v-tooltip activator="parent" location="bottom">Delete view</v-tooltip>
                    </IconButton>
                </template>
            </ListItem>
        </template>
        <template #additional-filter>
            <FilterDropdown
                v-model="templateIds"
                :item-manager="itemManager"
                :mapper="(item) => item.filterByTemplate.nodes"
                label="Template"
                :fetch-on-search="templateFetch"
            />
        </template>
        <CreateViewDialog :project="trackableId" :templates="templates" @created-view="modifiedViews.push($event.id)" />
        <UpdateViewDialog v-model="viewToUpdate" :templates="templates" @updated-view="modifiedViews.push($event.id)" />
    </PaginatedList>
</template>
<script lang="ts" setup>
import PaginatedList from "@/components/PaginatedList.vue";
import { NodeReturnType, useClient } from "@/graphql/client";
import { ViewOrder, ViewOrderField } from "@/graphql/generated";
import { useRoute } from "vue-router";
import ListItem from "@/components/ListItem.vue";
import { computed, inject, ref } from "vue";
import CreateViewDialog from "@/components/dialog/CreateViewDialog.vue";
import { trackableKey } from "@/util/keys";
import { withErrorMessage } from "@/util/withErrorMessage";
import { computedAsync } from "@vueuse/core";
import ConfirmationDialog from "@/components/dialog/ConfirmationDialog.vue";
import UpdateViewDialog from "@/components/dialog/UpdateViewDialog.vue";
import { ItemManager } from "@/util/itemManager";
import { useFilterOption } from "@/util/useFilterOption";
import FilterDropdown from "@/components/input/FilterDropdown.vue";

type View = NodeReturnType<"getViewList", "Project">["views"]["nodes"][0];

const client = useClient();
const route = useRoute();
const trackable = inject(trackableKey);
const trackableId = computed(() => route.params.trackable as string);
const modifiedViews = ref<string[]>([]);
const viewToUpdate = ref<
    | {
          id: string;
          name: string;
          description: string;
          filterByTemplate: string[];
      }
    | undefined
>();

const sortFields = {
    Name: ViewOrderField.Name,
    "[Default]": ViewOrderField.Id
};

const templateIds = useFilterOption("template", true);
const templateInput = computed(() => {
    if (templateIds.value.length === 0) {
        return undefined;
    } else {
        return { any: { id: { in: templateIds.value } } };
    }
});
const templateFetch = async (query: string) =>
    client.searchComponentTemplates({ query: query, count: 100 }).then((res) => res.searchComponentTemplates);

const dependencyArray = computed(() => [modifiedViews, templateInput]);

class ViewItemManager extends ItemManager<View, ViewOrderField> {
    protected async fetchItems(
        filter: string | undefined,
        orderBy: ViewOrder[],
        count: number,
        page: number
    ): Promise<[View[], number]> {
        if (filter == undefined) {
            const res = (
                await client.getViewList({
                    orderBy,
                    count,
                    skip: page * count,
                    project: trackableId.value,
                    filter: { filterByTemplate: templateInput.value }
                })
            ).node as NodeReturnType<"getViewList", "Project">;
            return [res.views.nodes!, res.views.totalCount];
        } else {
            const res = await client.getFilteredViewList({
                query: filter,
                count,
                filter: { filterByTemplate: templateInput.value, project: { id: { eq: trackableId.value } } }
            });
            return [res.searchViews, res.searchViews.length];
        }
    }
}
const itemManager: ItemManager<View, ViewOrderField> = new ViewItemManager();

const templates = computedAsync(async () => {
    return withErrorMessage(async () => {
        const project = (await client.getProjectComponentTemplates({ project: trackableId.value }))
            .node as NodeReturnType<"getProjectComponentTemplates", "Project">;
        const templateLookup = new Map<string, { id: string; name: string }>();
        for (const componentVersion of project.components.nodes) {
            const template = componentVersion.component.template;
            templateLookup.set(template.id, { id: template.id, name: template.name });
        }
        const templates = [...templateLookup.values()];
        templates.sort((a, b) => a.name.localeCompare(b.name));
        return templates;
    }, "Error loading templates");
}, []);

const manageViews = computed(() => (trackable!.value as { manageViews: boolean } | undefined)?.manageViews ?? false);

function updateView(view: View) {
    viewToUpdate.value = {
        id: view.id,
        name: view.name,
        description: view.description,
        filterByTemplate: view.filterByTemplate.nodes.map((template) => template.id)
    };
}

async function deleteView(viewId: string) {
    withErrorMessage(async () => {
        await client.deleteView({ id: viewId });
    }, "Error deleting view");
    modifiedViews.value.push(viewId);
}
</script>
<style scoped lang="scss">
@use "@/styles/settings";
.issue-container {
    min-width: settings.$icon-with-number-width;
}

.chip-container {
    row-gap: 0.5rem;
    column-gap: 0.25rem;
}
</style>
