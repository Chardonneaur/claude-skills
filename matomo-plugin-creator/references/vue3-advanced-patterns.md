# Vue 3 Advanced Patterns Reference

Advanced Vue 3 patterns for Matomo plugin development: TypeScript, stores, CoreHome imports,
form handling, and API integration. Based on real patterns from CoreHome, CorePluginsAdmin,
TagManager, and other core plugins.

## Table of Contents

1. [CoreHome Imports Reference](#corehome-imports-reference)
2. [CorePluginsAdmin Imports](#corepluginadmin-imports)
3. [Singleton Store Pattern](#singleton-store-pattern)
4. [API Calls with AjaxHelper](#api-calls-with-ajaxhelper)
5. [Component Structure with TypeScript](#component-structure-with-typescript)
6. [Form Handling with Field Component](#form-handling-with-field-component)
7. [Cross-Plugin Component Loading](#cross-plugin-component-loading)
8. [Entry Point Pattern](#entry-point-pattern)
9. [Type Definitions](#type-definitions)
10. [URL State Management](#url-state-management)
11. [Notifications](#notifications)

---

## CoreHome Imports Reference

Import from `'CoreHome'` in your Vue components. These are the most commonly used exports:

```typescript
// Utilities
import { translate, translateOrDefault } from 'CoreHome';
import { AjaxHelper, AjaxOptions } from 'CoreHome';
import { default as MatomoUrl } from 'CoreHome';
import { Matomo } from 'CoreHome';
import { default as debounce } from 'CoreHome';
import { default as clone } from 'CoreHome';

// Components
import { ActivityIndicator } from 'CoreHome';
import { Alert } from 'CoreHome';
import { ContentBlock } from 'CoreHome';
import { EnrichedHeadline } from 'CoreHome';
import { MatomoDialog } from 'CoreHome';
import { SiteSelector } from 'CoreHome';
import { DatePicker } from 'CoreHome';
import { PeriodSelector } from 'CoreHome';
import { Sparkline } from 'CoreHome';
import { Progressbar } from 'CoreHome';
import { WidgetLoader } from 'CoreHome';
import { FieldArray } from 'CoreHome';
import { MultiPairField } from 'CoreHome';

// Stores
import { SitesStore } from 'CoreHome';
import { ReportingMenuStore } from 'CoreHome';
import { ReportMetadataStore } from 'CoreHome';
import { WidgetsStore } from 'CoreHome';
import { NotificationsStore } from 'CoreHome';
import { ComparisonsStore } from 'CoreHome';

// Directives
import { FocusAnywhereButHere } from 'CoreHome';
import { FocusIf } from 'CoreHome';
import { Tooltips } from 'CoreHome';
import { ExpandOnClick } from 'CoreHome';
import { CopyToClipboard } from 'CoreHome';
import { ShowSensitiveData } from 'CoreHome';
import { SelectOnFocus } from 'CoreHome';

// Cookie helpers
import { setCookie, getCookie, deleteCookie } from 'CoreHome';

// Cross-plugin component loading
import { default as useExternalPluginComponent } from 'CoreHome';

// Periods
import { Periods } from 'CoreHome';
```

---

## CorePluginsAdmin Imports

Import from `'CorePluginsAdmin'` for form and settings UI components:

```typescript
import { Form } from 'CorePluginsAdmin';
import { Field } from 'CorePluginsAdmin';
import { FormField } from 'CorePluginsAdmin';
import { SaveButton } from 'CorePluginsAdmin';
```

---

## Singleton Store Pattern

Matomo uses a class-based singleton pattern with Vue 3 reactivity:

```typescript
// vue/src/MyStore/MyStore.store.ts

import { reactive, readonly, computed, DeepReadonly } from 'vue';
import { AjaxHelper } from 'CoreHome';

interface MyStoreState {
    items: MyItem[];
    isLoading: boolean;
    isInitialized: boolean;
}

interface MyItem {
    id: number;
    name: string;
    status: string;
}

class MyStore {
    // Private reactive state — only mutations happen here
    private privateState = reactive<MyStoreState>({
        items: [],
        isLoading: false,
        isInitialized: false,
    });

    // Public readonly state — prevents external mutations
    get state(): DeepReadonly<MyStoreState> {
        return readonly(this.privateState);
    }

    // Computed properties for derived data
    readonly items = computed(() => this.state.items);
    readonly isLoading = computed(() => this.state.isLoading);
    readonly activeItems = computed(() =>
        this.state.items.filter(item => item.status === 'active')
    );

    // Promise caching to prevent duplicate requests
    private fetchPromise: Promise<MyItem[]> | null = null;

    // Fetch with caching
    fetchItems(idSite: number): Promise<MyItem[]> {
        if (this.fetchPromise) {
            return this.fetchPromise;
        }

        this.privateState.isLoading = true;

        this.fetchPromise = AjaxHelper.fetch<MyItem[]>({
            method: 'MyPlugin.getItems',
            idSite: idSite,
            filter_limit: '-1',
        }).then((items) => {
            this.privateState.items = items;
            this.privateState.isInitialized = true;
            return items;
        }).finally(() => {
            this.privateState.isLoading = false;
        });

        return this.fetchPromise;
    }

    // Find with fallback to API
    findItem(idSite: number, itemId: number): Promise<DeepReadonly<MyItem>> {
        const found = this.items.value.find(item => item.id === itemId);
        if (found) {
            return Promise.resolve(found);
        }

        return AjaxHelper.fetch<MyItem>({
            method: 'MyPlugin.getItem',
            idSite,
            itemId,
        }).then((item) => {
            this.privateState.items = [...this.privateState.items, item];
            return readonly(item);
        });
    }

    // Mutation method
    reload(): void {
        this.fetchPromise = null;
        this.privateState.isInitialized = false;
    }
}

// Export singleton instance
export default new MyStore();
```

---

## API Calls with AjaxHelper

### GET requests (read data)

```typescript
import { AjaxHelper } from 'CoreHome';

// Simple fetch
const data = await AjaxHelper.fetch<MyItem[]>({
    method: 'MyPlugin.getItems',
    idSite: 1,
    filter_limit: '-1',
});

// With additional options
const data = await AjaxHelper.fetch<MyItem>({
    method: 'MyPlugin.getItem',
    idSite: 1,
    itemId: 42,
}, {
    createErrorNotification: true,  // Show error notification on failure
});
```

### POST requests (write data)

```typescript
import { AjaxHelper } from 'CoreHome';

// POST with token authentication (required for write operations)
const result = await AjaxHelper.post<{ value: number }>(
    { method: 'MyPlugin.addItem' },              // URL params
    { name: 'New Item', idSite: 1 },              // POST body
    { withTokenInUrl: true },                      // Options
);
```

### Abort controller pattern

```typescript
// Create a one-at-a-time request (cancels previous on new call)
const fetchItems = AjaxHelper.oneAtATime<MyItem[]>('MyPlugin.getItems');

// Each call cancels the previous one
const items = await fetchItems({ idSite: 1, filter_limit: '10' });
```

---

## Component Structure with TypeScript

```vue
<!-- vue/src/MyComponent/MyComponent.vue -->
<template>
    <div class="myComponent">
        <ContentBlock :content-title="translate('MyPlugin_Title')">
            <ActivityIndicator :loading="isLoading" />

            <div v-if="!isLoading && items.length === 0">
                {{ translate('CoreHome_NoData') }}
            </div>

            <table v-if="items.length > 0" class="entityTable">
                <thead>
                    <tr>
                        <th>{{ translate('General_Name') }}</th>
                        <th>{{ translate('General_Actions') }}</th>
                    </tr>
                </thead>
                <tbody>
                    <tr v-for="item in items" :key="item.id">
                        <td>{{ item.name }}</td>
                        <td>
                            <button @click="deleteItem(item.id)">
                                {{ translate('General_Delete') }}
                            </button>
                        </td>
                    </tr>
                </tbody>
            </table>
        </ContentBlock>
    </div>
</template>

<script lang="ts">
import { defineComponent, ref, computed, onMounted } from 'vue';
import { translate, AjaxHelper, ContentBlock, ActivityIndicator } from 'CoreHome';
import MyStore from '../MyStore/MyStore.store';

interface MyItem {
    id: number;
    name: string;
}

export default defineComponent({
    components: {
        ContentBlock,
        ActivityIndicator,
    },
    props: {
        idSite: {
            type: Number,
            required: true,
        },
    },
    setup(props) {
        const isLoading = computed(() => MyStore.isLoading.value);
        const items = computed(() => MyStore.items.value);

        onMounted(() => {
            MyStore.fetchItems(props.idSite);
        });

        const deleteItem = async (id: number) => {
            await AjaxHelper.post(
                { method: 'MyPlugin.deleteItem' },
                { idSite: props.idSite, itemId: id },
                { withTokenInUrl: true },
            );
            MyStore.reload();
            MyStore.fetchItems(props.idSite);
        };

        return {
            translate,
            isLoading,
            items,
            deleteItem,
        };
    },
});
</script>
```

---

## Form Handling with Field Component

The `Field` component from `CorePluginsAdmin` handles all standard form field types:

```vue
<template>
    <div>
        <Field
            v-model="formData.name"
            :title="translate('General_Name')"
            :uicontrol="'text'"
            name="name"
        />

        <Field
            v-model="formData.description"
            :title="translate('General_Description')"
            :uicontrol="'textarea'"
            name="description"
        />

        <Field
            v-model="formData.enabled"
            :title="translate('MyPlugin_Enabled')"
            :uicontrol="'checkbox'"
            name="enabled"
        />

        <Field
            v-model="formData.type"
            :title="translate('MyPlugin_Type')"
            :uicontrol="'select'"
            name="type"
            :options="typeOptions"
        />

        <Field
            v-model="formData.sites"
            :title="translate('MyPlugin_Sites')"
            :uicontrol="'multiselect'"
            name="sites"
            :options="siteOptions"
        />

        <Field
            v-model="formData.idSite"
            :title="translate('MyPlugin_Website')"
            :uicontrol="'site'"
            name="idSite"
        />

        <SaveButton
            @confirm="save"
            :saving="isSaving"
        />
    </div>
</template>

<script lang="ts">
import { defineComponent, reactive, ref } from 'vue';
import { translate, AjaxHelper } from 'CoreHome';
import { Field, SaveButton } from 'CorePluginsAdmin';

export default defineComponent({
    components: { Field, SaveButton },
    setup() {
        const formData = reactive({
            name: '',
            description: '',
            enabled: true,
            type: 'default',
            sites: [],
            idSite: 1,
        });

        const isSaving = ref(false);

        const typeOptions = [
            { key: 'default', value: 'Default' },
            { key: 'custom', value: 'Custom' },
        ];

        const save = async () => {
            isSaving.value = true;
            try {
                await AjaxHelper.post(
                    { method: 'MyPlugin.save' },
                    { ...formData },
                    { withTokenInUrl: true },
                );
            } finally {
                isSaving.value = false;
            }
        };

        return { translate, formData, isSaving, typeOptions, save };
    },
});
</script>
```

### Available uicontrol values

| Value | Description |
|-------|-------------|
| `'text'` | Single-line text input |
| `'textarea'` | Multi-line text area |
| `'checkbox'` | Boolean checkbox |
| `'select'` | Single-select dropdown |
| `'multiselect'` | Multi-select dropdown |
| `'site'` | Matomo site selector |
| `'password'` | Password input |
| `'number'` | Numeric input |
| `'radio'` | Radio button group |

---

## Cross-Plugin Component Loading

Load components from other plugins dynamically:

```typescript
import { useExternalPluginComponent } from 'CoreHome';

// In setup() or computed:
const ExternalComponent = useExternalPluginComponent('TagManager', 'ContainerList');

// In template:
// <component :is="ExternalComponent" v-bind="props" />
```

This uses `defineAsyncComponent` + dynamic UMD script loading internally.

---

## Entry Point Pattern

Every plugin's Vue module needs an `index.ts` that exports all public components:

```typescript
// vue/src/index.ts

export { default as MyComponent } from './MyComponent/MyComponent.vue';
export { default as MyOtherComponent } from './MyOtherComponent/MyOtherComponent.vue';
export { default as MyStore } from './MyStore/MyStore.store';
export * from './types';
```

### Using exported components in Twig

```twig
{# Load via vue-entry attribute #}
<div vue-entry="MyPlugin.MyComponent"
     id-site="{{ idSite }}"
     some-prop="{{ someValue|e('html_attr') }}">
</div>
```

**Important:** Props are passed as kebab-case HTML attributes. Arrays and objects must be
JSON-encoded in the controller:

```php
// In Controller
$view->someData = json_encode($data);
```

```twig
<div vue-entry="MyPlugin.MyComponent"
     some-data="{{ someData|e('html_attr') }}">
</div>
```

---

## Type Definitions

Define shared TypeScript interfaces in a `types.ts` file:

```typescript
// vue/src/types.ts

export interface MyItem {
    id: number;
    idSite: number;
    name: string;
    description: string;
    status: 'active' | 'inactive' | 'deleted';
    createdDate: string;
    parameters?: Record<string, unknown>;
}

export interface MyItemCreateRequest {
    idSite: number;
    name: string;
    description?: string;
}

export interface PaginationState {
    currentPage: number;
    pageSize: number;
    totalItems: number;
}
```

---

## URL State Management

Use `MatomoUrl` for reactive URL parameter access:

```typescript
import { MatomoUrl } from 'CoreHome';
import { watch, computed } from 'vue';

// Read current URL parameters (reactive)
const idSite = computed(() => MatomoUrl.parsed.value.idSite);
const period = computed(() => MatomoUrl.parsed.value.period);
const date = computed(() => MatomoUrl.parsed.value.date);
const segment = computed(() => MatomoUrl.parsed.value.segment);

// Watch for URL changes
watch(
    () => MatomoUrl.parsed.value.category,
    (newCategory) => {
        // React to navigation changes
        loadData(newCategory);
    },
    { immediate: true }
);

// Update URL parameters
MatomoUrl.updateHash({
    category: 'MyPlugin_Category',
    subcategory: 'mySubpage',
});
```

---

## Notifications

Show user notifications from Vue components:

```typescript
import { NotificationsStore } from 'CoreHome';

// Success notification
NotificationsStore.show({
    message: translate('MyPlugin_SavedSuccessfully'),
    context: 'success',
    type: 'transient',  // Auto-dismisses
    id: 'MyPlugin_SaveNotification',
});

// Error notification
NotificationsStore.show({
    message: translate('MyPlugin_SaveFailed'),
    context: 'error',
    type: 'persistent',  // Stays until dismissed
    id: 'MyPlugin_ErrorNotification',
});

// Toast notification (floating)
NotificationsStore.toast({
    message: translate('MyPlugin_ItemDeleted'),
    context: 'success',
    type: 'toast',
    id: 'MyPlugin_DeleteToast',
});

// Remove a specific notification
NotificationsStore.remove('MyPlugin_SaveNotification');
```

### Notification types

| Type | Behavior |
|------|----------|
| `'transient'` | Auto-dismisses after a delay |
| `'persistent'` | Stays until user dismisses or code removes it |
| `'toast'` | Floating notification that auto-dismisses |

### Notification contexts

| Context | Appearance |
|---------|------------|
| `'success'` | Green |
| `'error'` | Red |
| `'warning'` | Yellow |
| `'info'` | Blue |
