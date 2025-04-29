<template>
  <component :dusk="field.attribute" :is="field.fullWidth ? 'FullWidthField' : 'default-field'" :field="field"
    :errors="errors" :show-help-text="showHelpText" full-width-content>
    <template #field>

      <div class="flex flex-row gap-4">
        <div class="w-1/2" v-if="searchable">
          <input v-model="tempSearchName" @input="onInputChange" placeholder="Search by course/lesson name or label"
            type="text" class="w-full form-control form-input form-control-bordered search-style" />
        </div>
        <div class="w-1/2" v-if="hasActiveTermFilter">
          <Multiselect v-model="tempSelectedTerm" @change="onTermChange" :options="availableTerms" placeholder="Filter by Active Term"
            track-by="value" :searchable="true" />
        </div>
      </div>

      <!-- Main groups with properly positioned overlay -->
      <div ref="flexibleFieldContainer" class="flexible-field-content">
        <!-- Global spinner overlay for search/filter operations - only covers content area -->
        <div v-if="isFiltering" class="filter-loading-overlay">
          <div class="filter-spinner-container">
            <svg class="animate-spin" width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
              <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
              <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
            </svg>
          </div>
        </div>

          <form-nova-flexible-content-group 
            v-for="(group, groupIndex) in visibleGroups"
            :key="group.key" 
            :dusk="field.attribute + '-' + groupIndex" 
            :field="field" 
            :group="group" 
            :index="groupIndex"
            :deleteLoading="deleteLoading"
            :resource-name="resourceName" 
            :resource-id="resourceId" 
            :errors="errors" 
            :mode="mode"
            :order="order"
            @move-up="moveUp(group.key)" 
            @move-down="moveDown(group.key)" 
            @remove="remove(group.key)" 
            :draggable="!showLoadMoreButton" 
            :moveDownStatus="!(showLoadMoreButton && groupIndex === visibleGroups.length - 1) && (groupIndex !== filteredGroupsFull.length - 1)"
            />
        
        <!-- Load more button -->
        <div class="load-more-container" v-if="isLoading">
          <div class="spinner-grow text-primary" role="status" >
            <span class="sr-only"></span>
          </div>
        </div>
        <div class="load-more-container" v-if="showLoadMoreButton">
          <button v-if="!isLoading" @click="loadMore" class="btn btn-default btn-primary opacity-75 cursor-not-allowed" >
            <span  class="spinner-border spinner-border-sm mr-2" role="status" aria-hidden="true"></span>
            Load More
          </button>
          <hr class="last-divider mb-3" />
        </div>

        <!-- Last groups -->
          <form-nova-flexible-content-group 
            v-for="(group, groupIndex) in lastGroups"
            :key="'last-' + group.key" 
            :dusk="field.attribute + '-' + (filteredGroupsFull.length - initialGroupsCount) + groupIndex" 
            :draggable="!showLoadMoreButton" 
            :field="field" 
            :group="group" 
            :deleteLoading="deleteLoading"
            :index="(filteredGroupsFull.length - initialGroupsCount) + groupIndex"
            :resource-name="resourceName" 
            :resource-id="resourceId" 
            :errors="errors" 
            :mode="mode"
            :order="order"
            :moveUpStatus="!showLoadMoreButton || groupIndex !== 0"   
            :moveDownStatus="!showLoadMoreButton || (lastGroups.length > 1 && groupIndex !== lastGroups.length - 1)" 
            @move-up="moveUp(group.key)" 
            @move-down="moveDown(group.key)"  
            @remove="remove(group.key)" 
          />
      </div>

      <component :layouts="layouts" :is="field.menu.component" :field="field" :limit-counter="limitCounter"
        :limit-per-layout-counter="limitPerLayoutCounter" :errors="errors" :resource-name="resourceName"
        :resource-id="resourceId" @addGroup="addGroup($event)" />
    </template>
  </component>
</template>

<script>
import FullWidthField from "./FullWidthField";
import Sortable from "sortablejs";
import { DependentFormField, HandlesValidationErrors, mapProps, } from "laravel-nova";
import Group from "../group";
import Multiselect from "@vueform/multiselect";

export default {
  mixins: [HandlesValidationErrors, DependentFormField],

  inheritAttrs: false,

  props: {
    ...mapProps(["resourceName", "resourceId", "mode"]),
  },

  components: { FullWidthField, Multiselect },

  data() {
    return {
      order: [],
      groups: {},
      files: {},
      initialGroupsCount: 1,
      sortableInstance: null,
      visibleCount: this.field.initialItemsCount !== null
      ? Math.max(this.field.initialItemsCount - 1, 1)
      : null,
      deleteLoading: false,
      selectedTerm: null,
      tempSelectedTerm: null, // Temporary holder before updating the reactive property
      availableTerms: [
        { value: "ALL", label: "All" },
        { value: "FIRST_TERM", label: "First Term" },
        { value: "SECOND_TERM", label: "Second Term" }
      ],
      searchable: this.field.searchable || null,
      searchName: '',
      tempSearchName: '', // Temporary holder before updating the reactive property
      isLoading: false,
      isFiltering: false, // Track if filtering/searching is in progress
    };
  },

  computed: {
    layouts() {
      return this.field.layouts || false;
    },

    orderedGroups() {
      return this.order.reduce((groups, key) => {
        groups.push(this.groups[key]);
        return groups;
      }, []);
    },
    filteredGroupsFull() {
      // Start with all ordered groups
      let filteredGroups = this.orderedGroups;
      
      // First filter by term if term filter is active
      if (this.hasActiveTermFilter && this.selectedTerm) {
        filteredGroups = filteredGroups.filter(group => this.matchesSelectedTerm(group));
      }
      
      // Then filter by search name if search is active
      if (this.searchable && this.searchName) {
        filteredGroups = filteredGroups.filter(group => this.matchesSearchName(group));
      }

      return filteredGroups;
    },
    visibleGroups() {
      // Use the main end index calculation regardless of filtering
      const mainEndIndex = this.lastGroups ? 
        this.filteredGroupsFull.length - this.initialGroupsCount : 
        this.filteredGroupsFull.length;
      
      return this.filteredGroupsFull.slice(0, Math.min(this.visibleCount, mainEndIndex));
    },
    lastGroups() {
      const groups = this.filteredGroupsFull;
      if (!this.paginate() || groups.length <= this.visibleCount) {
        return null;
      }
      return groups.slice(-this.initialGroupsCount);
    },

    showLoadMoreButton() {
      return this.paginate() && (this.filteredGroupsFull.length > this.visibleCount + this.initialGroupsCount);
    },
    computedMoveStatus() {
    return !this.showLoadMoreButton;
    },
    limitCounter() {
      if (
        this.field.limit === null ||
        typeof this.field.limit == "undefined"
      ) {
        return null;
      }

      return this.field.limit - Object.keys(this.groups).length;
    },
    limitPerLayoutCounter() {
      return this.layouts.reduce((layoutCounts, layout) => {
        if (layout.limit === null) {
          layoutCounts[layout.name] = null;

          return layoutCounts;
        }
        let count = Object.values(this.groups).filter(
          (group) => group.name === layout.name,
        ).length;

        layoutCounts[layout.name] = layout.limit - count;

        return layoutCounts;
      }, {});
    },
   
    hasActiveTermFilter() {
      return this.field.hasActiveTermFilter;
    }
  },
  watch: {
    searchName() {
      // Always show spinner when search changes - immediately
      this.isFiltering = true;
      
      // Release thread to ensure spinner appears before heavy work
      setTimeout(() => {
        this.emitButtonState();
        
        // Keep spinner visible for longer on large datasets
        setTimeout(() => {
          this.isFiltering = false;
        }, 800);
      }, 0);
    },
    selectedTerm() {
      // Always show spinner when term selection changes - immediately
      this.isFiltering = true;
      
      // Release thread to ensure spinner appears before heavy work
      setTimeout(() => {
        this.emitButtonState();
        
        // Keep spinner visible for longer on large datasets
        setTimeout(() => {
          this.isFiltering = false;
        }, 800);
      }, 0);
    },
    
  },

  methods: {
    paginate() {
      return this.field.paginate;
    },
    onSearchKeyUp() {
      // This is handled by onInputChange now
    },
    _computeFilteredGroups() {
      this.$forceUpdate();
    },
    emitButtonState() {
      this.visibleCount = this.field.initialItemsCount !== null
      ? Math.max(this.field.initialItemsCount - 1, 1)
      : null
      if (this.searchName == "" && this.selectedTerm == null) {
        Nova.$emit('set-button-state', false);
      } else {        
        Nova.$emit('set-button-state', true);
      }
    },
    loadMore() {
      // Start loading state
      this.isLoading = true;
      
      const batchSize = this.field.paginationCount || 10;
      const startIndex = this.visibleCount;
      const endIndex = Math.min(
        startIndex + batchSize,
        this.orderedGroups.length - this.initialGroupsCount
      );
      
      // IMMEDIATE UPDATE: Show the first item immediately for better UX
      // This gives immediate feedback that something is happening
      this.visibleCount = startIndex + 1;
      
      // Give the browser a moment to render the first item
      this.$nextTick(() => {
        // If we only had one item to load, we're done
        if (startIndex + 1 >= endIndex) {
          this.isLoading = false;
          return;
        }
        
        // Process the rest of the items in smaller batches
        // Starting from the second item (we already showed the first)
        setTimeout(() => {
          this.processNextBatch(startIndex, endIndex, 1);
        }, 50);
      });
    },
    
    processNextBatch(startIndex, endIndex, currentIndex) {
      // Process a small batch at a time to keep the UI responsive
      const batchSize = 1; // Process 1 items at once
      const nextIndex = Math.min(currentIndex + batchSize, endIndex - startIndex);
      
      // Update visibleCount to show processed items
      this.visibleCount = startIndex + nextIndex;
      
      if (nextIndex < endIndex - startIndex) {
        // Schedule next batch after a short delay
        setTimeout(() => {
          this.processNextBatch(startIndex, endIndex, nextIndex);
        }, 50);
      } else {
        // We're done loading all requested items
        this.$nextTick(() => {
          this.isLoading = false;
        });
      }
    },
    matchesSelectedTerm(group) {
      const term = this.extractGroupTerm(group);
      return term === this.selectedTerm;
    },

    extractGroupTerm(group) {
      return group.fields.find(
        f => f.attribute.endsWith('__course_heading_active_term') ||
          f.attribute.endsWith('__lesson_active_term')
      )?.value;
    },
    matchesSearchName(group) {
      const search = this.searchName.toLowerCase();
      const nameString = group.fields
        .filter(f => f.attribute.endsWith('__course_heading_name') ||
          f.attribute.endsWith('__lesson_name') ||
          f.attribute.endsWith('__lesson_label'))
        .map(f => f.value || '')
        .join(' ')
        .toLowerCase();
      return nameString.includes(search);
    },

    /*
    * Set the initial, internal value for the field.
    */
    setInitialValue() {
      this.value = this.currentField.value || [];
      this.files = {};
      this.populateGroups();
      this.$nextTick(this.initSortable.bind(this));
    },

    /**
     * Fill the given FormData object with the field's internal value.
     */
    fill(formData) {
      let key, group;

      this.value = [];
      this.files = {};
      
      for (var i = 0; i < this.order.length; i++) {
        key = this.order[i];
        
        const isShown = [...this.visibleGroups, ...(this.lastGroups || [])].some(group => group.key === key);
       
    
        if(!isShown){
          continue ;
        }
        group = this.groups[key].serialize();
        group.attributes[`${key}__order`] = i;
        // Only serialize the group's non-file attributes
        this.value.push({
          layout: group.layout,
          key: group.key,
          attributes: group.attributes,
          
        });
        
        // Attach the files for formData appending
        this.files = { ...this.files, ...group.files };
      }

      this.appendFieldAttribute(formData, this.currentField.attribute);
      formData.append(
        this.currentField.attribute,
        this.value.length ? JSON.stringify(this.value) : "",
      );

      // Append file uploads
      for (let file in this.files) {
        formData.append(file, this.files[file]);
      }

      this.$nextTick(this.initSortable.bind(this));
    },

    /**
     * Register given field attribute into the parsable flexible fields register
     */
    appendFieldAttribute(formData, attribute) {
      let registered = [];

      if (formData.has("___nova_flexible_content_fields")) {
        registered = JSON.parse(
          formData.get("___nova_flexible_content_fields"),
        );
      }

      registered.push(attribute);

      formData.set(
        "___nova_flexible_content_fields",
        JSON.stringify(registered),
      );
    },

    /**
     * Update the field's internal value.
     */
    handleChange(value) {
      this.value = value || [];
      this.files = {};
      
      this.populateGroups();
      
    },

    /**
     * Set the displayed layouts from the field's current value
     */
    populateGroups() {
      this.order.splice(0, this.order.length);
      this.groups = {};

      for (var i = 0; i < this.value.length; i++) {
        this.addGroup(
          this.getLayout(this.value[i].layout),
          this.value[i].attributes,
          this.value[i].key,
          this.currentField.collapsed,
          true
        );
      }
    },
   
    /**
     * Retrieve layout definition from its name
     */
    getLayout(name) {
      if (!this.layouts) return;
      return this.layouts.find((layout) => layout.name === name);
    },

    /**
     * Append the given layout to flexible content's list
     */
    addGroup(layout, attributes, key, collapsed, populate = false) {
      if (!layout) return;

      collapsed = collapsed || false;

      let fields = attributes || JSON.parse(JSON.stringify(layout.fields));


      let group = new Group(
          layout.name,
          layout.title,
          fields,
          this.currentField,
          key,
          collapsed,
        );

      this.groups[group.key] = group;
      this.order.push(group.key);
      if (this.paginate() && !populate) {
        this.initialGroupsCount++;
        if(!this.showLoadMoreButton){
          this.visibleCount++;
        }
      }
    },


    /**
     * Move a group up
     */
    moveUp(key) {
      let index = this.order.indexOf(key);
      if (index <= 0) return;
      this.order.splice(index - 1, 0, this.order.splice(index, 1)[0]);
    },

    /**
     * Move a group down
     */
    moveDown(key) {
      let index = this.order.indexOf(key);
      if (index < 0 || index >= this.order.length - 1) return;

      this.order.splice(index + 1, 0, this.order.splice(index, 1)[0]);
    },

    /**
     * Remove a group
     */
     remove(key) {
      this.deleteLoading = true;
      let index = this.order.indexOf(key);
      if (index < 0) return;
      if (this.paginate() && key.includes('-')) {
        const [id, layout] = key.split('-');

        Nova.request()
          .post('delete-layout', {
            id,
            layout,
          })
          .then(() => {
            if(this.paginate()){
              let isInLastGroups = this.lastGroups?.some(group => group.key === key);
              if(isInLastGroups && this.lastGroups?.length > 1){
                this.initialGroupsCount --;
              }
            }
           
            this.order.splice(index, 1);
            delete this.groups[key];
            this.deleteLoading = false;
            Nova.success('Group deleted successfully');

          })
          .catch((result) => {
            Nova.error(result.response?.data?.message || 'Failed to delete group from server');
            this.deleteLoading = false;
          });
      } else {
        // No request needed, just delete locally
        if(this.paginate()){
          if(this.lastGroups?.length > 1){
            this.initialGroupsCount --;
          }
        }
        this.order.splice(index, 1);
        delete this.groups[key];
        this.deleteLoading = false;
      }
     
    },
    initSortable() {
      const containerRef = this.$refs["flexibleFieldContainer"];

      if (!containerRef || this.sortableInstance) {
        return;
      }

      this.sortableInstance = Sortable.create(containerRef, {
        ghostClass: "nova-flexible-content-sortable-ghost",
        dragClass: "nova-flexible-content-sortable-drag",
        chosenClass: "nova-flexible-content-sortable-chosen",
        direction: "vertical",
        handle: ".nova-flexible-content-drag-button",
        scrollSpeed: 5,
        animation: 500,
        onEnd: (evt) => {
          const item = evt.item;
          const key = item.id;
          const oldIndex = evt.oldIndex;
          const newIndex = evt.newIndex;

          if (oldIndex === newIndex) return;
 
          const movedKey = this.order.splice(oldIndex, 1)[0];
          this.order.splice(newIndex, 0, movedKey);
          
          // if (newIndex < oldIndex) {
          //   this.moveUp(key);
          // } else if (newIndex > oldIndex) {
          //   this.moveDown(key);
          // }
       
        },
      });
    },
    onInputChange(event) {
      // Always show spinner immediately
      this.isFiltering = true;
      
      // Use a short timeout to ensure spinner is displayed first
      setTimeout(() => {
        // Then update the actual searchName which triggers computation
        this.searchName = this.tempSearchName;
        
        // Force a longer spinner display time
        setTimeout(() => {
          this.isFiltering = false;
        }, 1000);
      }, 50);
    },
    onTermChange(value) {
      // Always show spinner immediately
      this.isFiltering = true;
      
      // Use a short timeout to ensure spinner is displayed first
      setTimeout(() => {
        // Then update the actual selectedTerm which triggers computation
        this.selectedTerm = this.tempSelectedTerm;
        
        // Force a longer spinner display time
        setTimeout(() => {
          this.isFiltering = false;
        }, 1000);
      }, 50);
    },
  }
};
</script>

<style lang="scss">
@import "@vueform/multiselect/themes/default.scss";

.load-more-container {
  display: flex;
  justify-content: center;
  margin-top: 10px;
  margin-bottom: 10px;
  flex-direction: column;
  align-items: center;
  cursor: pointer;
  padding: 8px;
}

.search-style {
  height: 41px;
  position: relative;
  margin: 0 auto;
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  box-sizing: border-box;
  cursor: pointer;
  outline: none;
  border: var(--ms-border-width, 1px) solid var(--ms-border-color, #D1D5DB);
  border-radius: var(--ms-radius, 4px);
  background: var(--ms-bg, #FFFFFF);
  font-size: var(--ms-font-size, 1rem);
  min-height: calc(2* var(--ms-border-width, 1px) + var(--ms-font-size, 1rem)* var(--ms-line-height, 1.375) + 2* var(--ms-py, 0.5rem));
}

.flexible-field-content {
  position: relative; /* Required for absolute positioning of overlay */
  margin-top: 10px;
}

.filter-loading-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(255, 255, 255, 0.7);
  z-index: 1000;
  display: flex;
  justify-content: center;
  align-items: center;
}

.filter-spinner-container {
  display: flex;
  align-items: center;
  justify-content: center;
  color: var(--primary);
}

.animate-spin {
  animation: spin 1s linear infinite;
  color: #4299e1; /* Tailwind blue-500 */
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}
</style>
