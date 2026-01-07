<template>
    <PaginatedList
        name="interface definitions"
        :item-manager="itemManager"
        :sort-fields="sortFields"
        :to="() => undefined"
        :dependencies="dependencyArray"
        query-param-prefix=""
    >
        <template #item="{ item }">
            <ListItem
                :title="`${item.name} (v${item.version})`"
                :subtitle="item.description || 'No description provided'"
                :italic-subtitle="!item.description"
            >
                <template #append>
                    <v-chip
                        v-if="item.visibleSelfDefined || item.visibleDerivedBy.totalCount > 0"
                        color="primary"
                        size="small"
                        class="flex-shrink-0 mr-5"
                        >Visible</v-chip
                    >
                    <v-chip
                        v-if="item.invisibleSelfDefined || item.invisibleDerivedBy.totalCount > 0"
                        color="primary"
                        size="small"
                        class="flex-shrink-0 mr-5"
                        >Invisible</v-chip
                    >
                    <v-chip
                        v-if="item.visibleDerivedBy.totalCount > 0"
                        color="primary"
                        size="small"
                        class="flex-shrink-0 mr-5"
                        >Visibly derived</v-chip
                    >
                    <v-chip
                        v-if="item.invisibleDerivedBy.totalCount > 0"
                        color="primary"
                        size="small"
                        class="flex-shrink-0 mr-5"
                        >Invisibly derived</v-chip
                    >
                    <IconButton
                        :disabled="!(trackable?.admin ?? false)"
                        @click="interfaceDefinitionToUpdate = item"
                        class="mr-2"
                    >
                        <v-icon icon="mdi-pencil" />
                        <v-tooltip activator="parent" location="bottom">Edit label</v-tooltip>
                    </IconButton>
                    <IconButton
                        :disabled="
                            !(trackable?.admin ?? false) ||
                            item.visibleDerivedBy.totalCount > 0 ||
                            item.invisibleDerivedBy.totalCount > 0
                        "
                    >
                        <v-icon icon="mdi-delete" />
                        <ConfirmationDialog
                            :title="`Delete interface definition?`"
                            message="
                                Are you sure you want to delete the interface definition?
                                This will delete all adjacent relations, and potentially stop interfaces from propagating to other components.
                            "
                            confirm-text="Delete"
                            @confirm="
                                deleteInterfaceDefinition(
                                    item.interfaceSpecificationVersion,
                                    item.visibleSelfDefined,
                                    item.invisibleSelfDefined
                                )
                            "
                        />
                        <v-tooltip activator="parent" location="bottom">Delete interface definition</v-tooltip>
                    </IconButton>
                </template>
            </ListItem>
        </template>
        <template #additional-filter>
            <div class="d-flex ga-2 align-center h-100">
                <FilterDropdown
                    v-model="templateIds"
                    :item-manager="itemManager"
                    :mapper="(item) => item.template"
                    label="Template"
                    :fetch-on-search="templateFetch"
                />
                <FilterDropdown
                    v-model="interfaceSpecificationIds"
                    :item-manager="itemManager"
                    :mapper="(item) => item.interfaceSpecificationData"
                    label="Interface Specification"
                    :fetch-on-search="interfaceSpecificationFetch"
                />
            </div>
        </template>
        <AddInterfaceSpecificationVersionToComponentVersionDialog
            :component="componentId"
            :component-version="componentVersionId"
            :component-template="componentTemplateInfo?.template?.id"
            @added-interface-specification-version-to-component-version="modifiedIds.push($event.id)"
        />
        <EditInterfaceDefinitionDialog
            :update-interface-definition="updateInterfaceDefinition"
            :component-template="componentTemplateInfo?.template?.id"
            v-model="interfaceDefinitionToUpdate"
        />
    </PaginatedList>
</template>
<script lang="ts" setup>
import PaginatedList from "@/components/PaginatedList.vue";
import { NodeReturnType, useClient } from "@/graphql/client";
import {
    DefaultInterfaceDefinitionInfoFragment,
    InterfaceDefinitionFilterInput,
    InterfaceDefinitionOrder,
    InterfaceDefinitionOrderField,
    InterfaceSpecificationFilterInput,
    InterfaceSpecificationVersionFilterInput
} from "@/graphql/generated";
import { useRoute } from "vue-router";
import ListItem from "@/components/ListItem.vue";
import { computed, inject, ref } from "vue";
import AddInterfaceSpecificationVersionToComponentVersionDialog from "@/components/dialog/AddInterfaceSpecificationVersionToComponentVersionDialog.vue";
import { computedAsync } from "@vueuse/core";
import { withErrorMessage } from "@/util/withErrorMessage";
import { trackableKey } from "@/util/keys";
import ConfirmationDialog from "@/components/dialog/ConfirmationDialog.vue";
import EditInterfaceDefinitionDialog from "@/components/dialog/EditInterfaceDefinitionDialog.vue";
import { ItemManager } from "@/util/itemManager";
import { useFilterOption } from "@/util/useFilterOption";
import FilterDropdown from "@/components/input/FilterDropdown.vue";

type InterfaceDefinition = DefaultInterfaceDefinitionInfoFragment & {
    name: string;
    version: string;
    interfaceSpecification: string;
    interfaceSpecificationVersion: string;
    description?: string;
    interfaceSpecificationData: { id: string; name: string; description: string };
    template: { id: string; name: string; description: string };
};

const client = useClient();
const route = useRoute();
const componentVersionId = computed(() => route.params.version as string);
const componentId = computed(() => route.params.trackable as string);
const trackable = inject(trackableKey);
const modifiedIds = ref<string[]>([]);
const interfaceDefinitionToUpdate = ref<InterfaceDefinition | undefined>();

const sortFields = {
    Version: InterfaceDefinitionOrderField.InterfaceSpecificationVersionVersion,
    "[Default]": InterfaceDefinitionOrderField.Id
};

const interfaceSpecificationIds = useFilterOption("interfacespecification", true);
const interfaceSpecificationInput = computed(() => {
    if (interfaceSpecificationIds.value.length === 0) {
        return undefined;
    }
    return {
        in: interfaceSpecificationIds.value
    };
});
const interfaceSpecificationFetch = async (search: string) =>
    client
        .searchInterfaceSpecifications({
            query: search,
            count: 100,
            component: componentId.value
        })
        .then((res) =>
            res.searchInterfaceSpecifications.map((t) => ({
                id: t.id,
                name: t.name,
                description: t.description
            }))
        );
const templateIds = useFilterOption("template", true);
const templateInput = computed(() => {
    if (templateIds.value.length === 0) {
        return undefined;
    }
    return {
        id: { in: templateIds.value }
    };
});
const templateFetch = async (search: string) =>
    client
        .searchInterfaceSpecificationTemplates({
            query: search,
            count: 100
        })
        .then((res) =>
            res.searchInterfaceSpecificationTemplates.map((t) => ({
                id: t.id,
                name: t.name,
                description: t.description
            }))
        );

const dependencyArray = computed(() => [modifiedIds, templateInput, interfaceSpecificationInput])

class InterfaceDefinitionItemManager extends ItemManager<InterfaceDefinition, InterfaceDefinitionOrderField> {
    protected async fetchItems(
        filter: string,
        orderBy: InterfaceDefinitionOrder[],
        count: number,
        page: number
    ): Promise<[InterfaceDefinition[], number]> {
        if (filter == undefined) {
            const res = (
                await client.getInterfaceDefinitionList({
                    orderBy,
                    count,
                    skip: page * count,
                    componentVersion: componentVersionId.value,
                    filter: {
                        interfaceSpecificationVersion: {
                            interfaceSpecification: {
                                id: interfaceSpecificationInput.value,
                                template: templateInput.value
                            }
                        }
                    }
                })
            ).node as NodeReturnType<"getInterfaceDefinitionList", "ComponentVersion">;
            return [
                res.interfaceDefinitions.nodes.map((definition) => ({
                    ...definition,
                    name: definition.interfaceSpecificationVersion.interfaceSpecification.name,
                    description: definition.interfaceSpecificationVersion.interfaceSpecification.description,
                    version: definition.interfaceSpecificationVersion.version,
                    interfaceSpecificationVersion: definition.interfaceSpecificationVersion.id,
                    interfaceSpecification: definition.interfaceSpecificationVersion.interfaceSpecification.id,
                    interfaceSpecificationData: definition.interfaceSpecificationVersion.interfaceSpecification,
                    template: definition.interfaceSpecificationVersion.interfaceSpecification.template
                })),
                res.interfaceDefinitions.totalCount
            ];
        } else {
            const definitionFilter: InterfaceDefinitionFilterInput = {
                componentVersion: { id: { eq: componentVersionId.value } }
            };
            const versionFilter: InterfaceSpecificationVersionFilterInput = {
                interfaceDefinitions: { any: definitionFilter }
            };
            const specificationFilter: InterfaceSpecificationFilterInput = {
                versions: { any: versionFilter },
                template: templateInput.value,
                id: interfaceSpecificationInput.value
            };
            const res = await client.getFilteredInterfaceDefinitionList({
                query: filter,
                count,
                specificationFilter,
                versionFilter,
                definitionFilter
            });
            const definitions: InterfaceDefinition[] = [];
            for (const interfaceSpecification of res.searchInterfaceSpecifications) {
                for (const version of interfaceSpecification.versions.nodes) {
                    for (const definition of version.interfaceDefinitions.nodes) {
                        definitions.push({
                            ...definition,
                            name: interfaceSpecification.name,
                            description: interfaceSpecification.description,
                            version: version.version,
                            interfaceSpecificationVersion: version.id,
                            interfaceSpecification: interfaceSpecification.id,
                            interfaceSpecificationData: interfaceSpecification,
                            template: interfaceSpecification.template
                        });
                    }
                }
            }
            return [definitions, res.searchInterfaceSpecifications.length];
        }
    }
}
const itemManager = new InterfaceDefinitionItemManager() as ItemManager<
    InterfaceDefinition,
    InterfaceDefinitionOrderField
>;

const componentTemplateInfo = computedAsync(
    async () => {
        const templateRes = await withErrorMessage(async () => {
            return client.getComponentTemplateDetails({ id: componentId.value });
        }, "Error loading component template info");
        return templateRes.node as NodeReturnType<"getComponentTemplateDetails", "Component">;
    },
    null,
    { shallow: false }
);

async function deleteInterfaceDefinition(
    interfaceSpecificationVersionId: string,
    visible: boolean,
    invisible: boolean
) {
    await withErrorMessage(async () => {
        await client.removeInterfaceSpecificationVersionFromComponentVersion({
            input: {
                interfaceSpecificationVersion: interfaceSpecificationVersionId,
                componentVersion: componentVersionId.value,
                visible,
                invisible
            }
        });
    }, "Error deleting interface definition");
    modifiedIds.value.push(interfaceSpecificationVersionId);
}

async function updateInterfaceDefinition(id: string, visible: boolean, value: boolean) {
    await withErrorMessage(async () => {
        const wrappedValue = {
            visible: visible,
            invisible: !visible,
            componentVersion: componentVersionId.value,
            interfaceSpecificationVersion: id
        };
        if (value) {
            await client.addInterfaceSpecificationVersionToComponentVersion({
                input: wrappedValue
            });
        } else {
            await client.removeInterfaceSpecificationVersionFromComponentVersion({
                input: wrappedValue
            });
        }
        modifiedIds.value.push(id);
    }, "Error updating interface definition visibility");
    modifiedIds.value.push(id);
}
</script>
