<template>
    <PaginatedList
        name="IMSs"
        :item-manager="itemManager"
        :sort-fields="sortFields"
        :to="(ims: IMS) => imsRoute(ims)"
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
                :mapper="(item) => item.template"
                label="Template"
                :fetch-on-search="templateFetch"
            />
        </template>
        <CreateIMSDialog @created-ims="(ims: IdObject) => selectIMS(ims)" />
    </PaginatedList>
</template>
<script lang="ts" setup>
import PaginatedList from "@/components/PaginatedList.vue";
import { ClientReturnType, useClient } from "@/graphql/client";
import { RouteLocationRaw, useRouter } from "vue-router";
import ListItem from "@/components/ListItem.vue";
import CreateIMSDialog from "@/components/dialog/CreateIMSDialog.vue";
import { IdObject } from "@/util/types";
import { ImsFilterInput, ImsOrder } from "@/graphql/generated";
import { ImsOrderField } from "@/graphql/generated";
import SyncSelfAllowedSwitch from "@/components/input/SyncSelfAllowedSwitch.vue";
import { ItemManager } from "@/util/itemManager";
import { useFilterOption } from "@/util/useFilterOption";
import FilterDropdown from "@/components/input/FilterDropdown.vue";
import { computed } from "vue";

type IMS = ClientReturnType<"getIMSList">["imss"]["nodes"][0];

const client = useClient();
const router = useRouter();

const sortFields = {
    Name: ImsOrderField.Name,
    Template: [ImsOrderField.TemplateName, ImsOrderField.TemplateId],
    "[Default]": ImsOrderField.Id
};

const templateIds = useFilterOption("template", true);
const templateInput = computed(() => {
    if (templateIds.value.length === 0) {
        return undefined;
    }
    return { id: { in: templateIds.value } };
});
const templateFetch = async (query: string) =>
    client.searchIMSTemplates({ query: query, count: 100 }).then((res) => res.searchIMSTemplates);

const dependencyArray = computed(() => [templateInput]);

class IMSItemManager extends ItemManager<IMS, ImsOrderField> {
    protected async fetchItems(
        filter: string | undefined,
        orderBy: ImsOrder[],
        count: number,
        page: number
    ): Promise<[IMS[], number]> {
        const generalFilter: ImsFilterInput = {
            template: templateInput.value
        };
        if (filter == undefined) {
            const res = await client.getIMSList({
                orderBy,
                count,
                skip: page * count,
                filter: generalFilter
            });
            return [res.imss.nodes, res.imss.totalCount];
        } else {
            const res = await client.getFilteredIMSList({
                query: filter,
                count,
                filter: generalFilter
            });
            return [res.searchIMSs, res.searchIMSs.length];
        }
    }
}

const itemManager: ItemManager<IMS, ImsOrderField> = new IMSItemManager();

function selectIMS(ims: IdObject) {
    router.push(imsRoute(ims));
}

function imsRoute(ims: IdObject): RouteLocationRaw {
    return {
        name: "ims",
        params: {
            ims: ims.id
        }
    };
}
</script>
