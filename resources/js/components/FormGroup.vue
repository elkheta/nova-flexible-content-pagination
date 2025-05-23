<template>
  <div class="relative mb-4 pb-1" :id="group.key">
    <div class="w-full shrink">
      <div :class="titleStyle" v-if="group.title">
        <div class="h-8 leading-normal h-full flex items-center box-content" :class="{
          'border-b border-gray-200 dark:border-gray-700 ': !collapsed,
        }">
          <button dusk="expand-group" type="button"
            class="shrink-0 group-control btn border-r border-gray-200 dark:border-gray-700 w-8 h-8 flex justify-center items-center"
            :title="__('Expand')" @click.prevent="expand" v-if="collapsed">
            <Icon name="plus" type="micro" class="align-top" />
          </button>
          <button dusk="collapse-group" type="button"
            class="group-control btn border-r border-gray-200 dark:border-gray-700 w-8 h-8 flex justify-center items-center"
            :title="__('Collapse')" @click.prevent="collapse" v-else>
            <Icon name="minus" type="micro" class="align-top" />
          </button>

          <p class="text-80 grow px-4">
            <span class="mr-3 font-semibold">
              #{{ calculateOrderNumber   }}
            </span>
            {{ group.title }}
          </p>

          <div class="flex" v-if="!readonly">
            <button v-if="draggable" dusk="drag-group" type="button"
              class="group-control btn border-l border-gray-200 dark:border-gray-700 w-8 h-8 flex justify-center items-center nova-flexible-content-drag-button"
              :title="__('Drag')">
              <Icon name="selector" type="micro" class="align-top" />
            </button>
            <button dusk="move-up-group" type="button" v-if="moveUpStatus"
              class="group-control btn border-l border-gray-200 dark:border-gray-700 w-8 h-8 flex justify-center items-center"
              :title="__('Move up')" @click.prevent="moveUp">
              <Icon name="arrow-up" type="micro" class="align-top" />
            </button>
            <button dusk="move-down-group" v-if="moveDownStatus" type="button"
              class="group-control btn border-l border-gray-200 dark:border-gray-700 w-8 h-8 flex justify-center items-center"
              :title="__('Move down')" @click.prevent="moveDown">
              <Icon name="arrow-down" class="align-top" type="micro" />
            </button>
            <button dusk="delete-group" type="button"
              class="group-control btn border-l border-gray-200 dark:border-gray-700 w-8 h-8 flex justify-center items-center"
              :title="__('Delete')" @click.prevent="confirmRemove">
              <Icon name="trash" class="align-top" type="micro" />
            </button>
            <delete-flexible-content-group-modal v-if="removeMessage" @confirm="remove" @close="removeMessage = false"
              :message="field.confirmRemoveMessage" :yes="field.confirmRemoveYes" :no="field.confirmRemoveNo" :deleteLoading="deleteLoading"
               />
          </div>
        </div>
      </div>
      <div :class="containerStyle">
        <component v-for="(item, index) in filteredFields" :key="item.key || index" :is="'form-' + item.component"  
            :resource-name="resourceName" :resource-id="resourceId" :field="item" :errors="errors" :mode="mode"  
              :show-help-text="item.helpText != null"
            :class="{ 'remove-bottom-border': index == filteredFields.length - 1 }" />           
      </div>
    </div>
  </div>
</template>

<script>
import { mapProps } from "laravel-nova";
import { Icon } from "laravel-nova-ui";

export default {
  components: { Icon },
  props: {
    errors: {},
    group: {},
    index: {},
    field: {},
    order: {},
    deleteLoading: {},
    draggable: {
      type: Boolean,
      default: true,
    },
    moveUpStatus: {
      type: Boolean,
      default: true,
    },
    moveDownStatus: {
      type: Boolean,
      default: true,
    },

    ...mapProps(["resourceName", "resourceId", "mode", 'order']),
  },

  emits: ["move-up", "move-down", "remove"],

  data() {
    return {
      removeMessage: false, 
      collapsed: this.group.collapsed || false,
      readonly: this.group.readonly,
      wasEverExpanded:  !this.group.collapsed || false,
    };
  },

  beforeDestroy() {
    Nova.$off('set-button-state');
  },
  computed: {
    // Filter fields into regular fields and nested flexible content fields
    filteredFields() {
      return this.group.fields.filter(item => {
        // Always show regular components
        if (item.component && item.component !== 'nova-flexible-content') {
          return true;
        }
        
        // Show nova-flexible-content only if wasEverCollapsed is true
        if (item.component === 'nova-flexible-content' && this.wasEverExpanded) {
          return true;
        }
        
        return false;
      });
    },
    
    
    calculateOrderNumber() {
      const orderField = this.group.fields.find(f => f.attribute.endsWith('__order'));
      const orderValue = (orderField?.value == null) ? this.index : orderField.value;
      return Number(orderValue) + 1;
    },
    titleStyle() {
      let classes = [
        "border-t",
        "border-r",
        "border-l",
        "border-gray-200",
        "dark:border-gray-700",
        "rounded-t-lg",
      ];

      if (this.collapsed) {
        classes.push("border-b rounded-b-lg");
      }

      return classes;
    },
    
    containerStyle() {
      let classes = [
        "grow",
        "border-b",
        "border-r",
        "border-l",
        "border-gray-200",
        "dark:border-gray-700",
        "rounded-b-lg",
      ];

      if (!this.group.title) {
        classes.push("border-t");
        classes.push("rounded-tr-lg");
      }

      if (this.collapsed) {
        classes.push("hidden");
      }

      return classes;
    },
  },

  methods: {
   
    /**
     * Move this group up
     */
    moveUp() {
      this.$emit("move-up");
    },

    /**
     * Move this group down
     */
    moveDown() {
      this.$emit("move-down");
    },

    /**
     * Remove this group
     */
    remove() {
      this.$emit("remove");
    },

    /**
     * Confirm remove message
     */
    confirmRemove() {
      if (this.field.confirmRemove) {
        this.removeMessage = true;
      } else {
        this.remove();
      }
    },

    /**
     * Expand fields
     */
    expand() {
      this.collapsed = false;
      this.wasEverExpanded = true;
    },

    /**
     * Collapse fields
     */
    collapse() {
      this.collapsed = true;
    },
  },
};
</script>

<style>
.group-control:focus {
  outline: none;
}

.group-control:hover {
  color: rgb(var(--colors-primary-400));
}

.confirm-message {
  position: absolute;
  overflow: visible;
  right: 38px;
  bottom: 0;
  width: auto;
  border-radius: 4px;
  padding: 6px 7px;
  border: 1px solid #b7cad6;
  background-color: var(--20);
  white-space: nowrap;
}

[dir="rtl"] .confirm-message {
  right: auto;
  left: 35px;
}

.confirm-message .text-danger {
  color: #ee3f22;
}

.rounded-l {
  border-top-left-radius: 0.25rem;
  /* 4px */
  border-bottom-left-radius: 0.25rem;
  /* 4px */
}

.rounded-t-lg {
  border-top-right-radius: 0.5rem;
  /* 8px */
  border-top-left-radius: 0.5rem;
  /* 8px */
}

.rounded-b-lg {
  border-bottom-left-radius: 0.5rem;
  /* 8px */
  border-bottom-right-radius: 0.5rem;
  /* 8px */
}

.box-content {
  box-sizing: content-box;
}

.grow {
  flex-grow: 1;
}

.grow-0 {
  flex-grow: 0;
}

.shrink {
  flex-shrink: 1;
}

.shrink-0 {
  flex-shrink: 0;
}
</style>
