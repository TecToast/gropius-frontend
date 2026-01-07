<template>
    <PaginatedList
        name="components"
        :item-manager="itemManager"
        :sort-fields="sortFields"
        :to="(component: Component) => componentRoute(component)"
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
                    <div class="text-medium-emphasis issue-container">
                        <v-icon icon="mdi-alert-circle-outline" />
                        {{ item.openIssues.totalCount }}
                    </div>
                </template>
            </ListItem>
        </template>
        <template #additional-filter>
            <FilterDropdown
                v-model="templateIds"
                :item-manager="itemManager"
                :mapper="(item) => item.template"
                label="Template"
                :fetch-on-search="templateFetch"
            />
        </template>
        <CreateComponentDialog @created-component="(component: IdObject) => selectComponent(component)" />
    </PaginatedList>
</template>
<script lang="ts" setup>
import PaginatedList from "@/components/PaginatedList.vue";
import { ClientReturnType, useClient } from "@/graphql/client";
import { ComponentFilterInput, ComponentOrder, ComponentOrderField } from "@/graphql/generated";
import { RouteLocationRaw, useRouter } from "vue-router";
import ListItem from "@/components/ListItem.vue";
import CreateComponentDialog from "@/components/dialog/CreateComponentDialog.vue";
import { IdObject } from "@/util/types";
import { ItemManager } from "@/util/itemManager";
import FilterDropdown from "@/components/input/FilterDropdown.vue";
import { useFilterOption } from "@/util/useFilterOption";
import { computed } from "vue";

type Component = ClientReturnType<"getComponentList">["components"]["nodes"][0];

const client = useClient();
const router = useRouter();

const sortFields = {
    Name: ComponentOrderField.Name,
    Template: [ComponentOrderField.TemplateName, ComponentOrderField.TemplateId],
    "[Default]": ComponentOrderField.Id
};

const templateIds = useFilterOption("template", true);
const templateInput = computed(() => {
    if (templateIds.value.length == 0) {
        return undefined;
    }
    return { id: { in: templateIds.value } };
});
const templateFetch = async (query: string) =>
    client.searchComponentTemplates({ query: query, count: 100 }).then((res) => res.searchComponentTemplates);

const dependencyArray = computed(() => [templateInput])

class ComponentItemManager extends ItemManager<Component, ComponentOrderField> {
    protected async fetchItems(
        filter: string,
        orderBy: ComponentOrder[],
        count: number,
        page: number
    ): Promise<[Component[], number]> {
        const generalFilters: ComponentFilterInput = {
            template: templateInput.value
        };
        if (filter == undefined) {
            const res = await client.getComponentList({
                orderBy,
                count,
                skip: page * count,
                filter: generalFilters
            });
            return [res.components.nodes, res.components.totalCount];
        } else {
            const res = await client.getFilteredComponentList({
                query: filter,
                count,
                filter: generalFilters
            });
            return [res.searchComponents, res.searchComponents.length];
        }
    }
}
const itemManager: ItemManager<Component, ComponentOrderField> = new ComponentItemManager();

function selectComponent(component: IdObject) {
    router.push(componentRoute(component));
}

function componentRoute(component: IdObject): RouteLocationRaw {
    return {
        name: "component",
        params: {
            trackable: component.id
        }
    };
}
</script>
<style scoped lang="scss">
@use "@/styles/settings";
.issue-container {
    min-width: settings.$icon-with-number-width;
}
</style>
