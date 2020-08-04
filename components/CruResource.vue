<script>
import isEmpty from 'lodash/isEmpty';
import { createYaml } from '@/utils/create-yaml';
import { clone } from '@/utils/object';
import { SCHEMA } from '@/config/types';
import ResourceYaml from '@/components/ResourceYaml';
import Banner from '@/components/Banner';
import AsyncButton from '@/components/AsyncButton';

export default {
  components: {
    Banner,
    ResourceYaml,
    AsyncButton
  },

  props: {
    doneRoute: {
      type:     String,
      required: true
    },

    mode: {
      type:     String,
      required: true
    },

    resource: {
      type:     Object,
      required: true
    },

    subtypes: {
      type:    Array,
      default: null
    },

    selectedSubtype: {
      type:    String,
      default: null
    },

    validationPassed: {
      type:    Boolean,
      default: true
    }
  },

  data() {
    return {
      isCancelModal: false,
      showAsForm:    true,
      resourceYaml:  '',
      errors:        []
    };
  },

  computed: {
    showSubtypeSelection() {
      const { selectedSubtype, subtypes } = this;

      if (isEmpty(subtypes)) {
        return false;
      }

      if (!selectedSubtype) {
        return true;
      }

      return false;
    }
  },

  methods: {
    checkCancel(isCancel) {
      if (isCancel) {
        this.isCancelModal = true;
      } else {
        this.isCancelModal = false;
      }

      this.$modal.show('cancel-modal');
    },

    confirmCancel(isCancel) {
      this.errors = [];
      this.$modal.hide('cancel-modal');

      if (isCancel) {
        this.isCancelModal = false;
        this.$router.replace({
          name:   this.doneRoute,
          params: { resource: this.resource.type }
        });
      } else {
        this.isCancelModal = false;
        this.resourceYaml = null;
        this.showAsForm = true;
      }
    },

    showPreviewYaml() {
      const schemas = this.$store.getters['cluster/all'](SCHEMA);
      const { resource } = this;
      const clonedResource = clone(resource);

      const resourceYaml = createYaml(schemas, resource.type, clonedResource);

      this.resourceYaml = resourceYaml;
      this.showAsForm = false;
    },

    yamlCb(success, errors) {
      success ? this.done() : (this.errors = errors);
    },

    done() {
      if (!this.doneRoute) {
        return;
      }
      this.$router.replace({
        name:   this.doneRoute,
        params: { resource: this.resource.type }
      });
    }
  }
};
</script>

<template>
  <section>
    <form class="create-resource-container">
      <div v-if="showSubtypeSelection" class="subtypes-container">
        <slot name="subtypes">
          <div
            v-for="subtype in subtypes"
            :key="subtype.id"
            class="subtype-banner"
            :class="{ selected: subtype.id === selectedSubtype }"
            @click="$emit('selectType', subtype.id)"
          >
            <slot name="subtype-logo">
              <div class="subtype-logo round-image">
                <img
                  v-if="subtype.bannerImage"
                  src="subtype.bannerImage"
                  alt="${ resource.type }: ${ subtype.label }"
                />
                <div v-else-if="subtype.bannerAbbrv" class="banner-abbrv">
                  <span
                    v-if="$store.getters['i18n/exists'](subtype.bannerAbbrv)"
                  >{{ t(subtype.bannerAbbrv) }}</span>
                  <span v-else>{{ subtype.bannerAbbrv.slice(0, 1).toUpperCase() }}</span>
                </div>
                <div v-else>
                  {{ subtype.id.slice(0, 1).toUpperCase() }}
                </div>
              </div>
            </slot>
            <slot name="subtype-content">
              <div class="subtype-content">
                <div class="title">
                  <h5>
                    <span
                      v-if="$store.getters['i18n/exists'](subtype.label)"
                      v-html="t(subtype.label)"
                    ></span>
                    <span v-else>{{ subtype.label }}</span>
                  </h5>
                </div>
                <div class="description">
                  <span
                    v-if="$store.getters['i18n/exists'](subtype.description)"
                    v-html="t(subtype.description)"
                  ></span>
                  <span v-else>{{ subtype.description }}</span>
                </div>
              </div>
            </slot>
          </div>
        </slot>
      </div>
      <template v-if="showAsForm">
        <div v-if="selectedSubtype || !subtypes.length" class="resource-container">
          <slot name="define" />
        </div>
        <div class="controls-row">
          <slot name="form-footer">
            <button type="button" class="btn role-secondary" @click="$emit('cancel')">
              <t k="generic.cancel" />
            </button>
            <div>
              <button
                v-if="selectedSubtype || !subtypes.length"
                type="button"
                class="btn role-secondary"
                @click="showPreviewYaml"
              >
                <t k="cruResource.previewYaml" />
              </button>
              <AsyncButton
                v-if="!showSubtypeSelection"
                :disabled="!validationPassed"
                :action-label="mode==='edit' ? t('generic.save') : t('generic.create')"
                @click="$emit('finish', done)"
              />
            </div>
          </slot>
        </div>
      </template>

      <section v-else class="cru-resource-yaml-container">
        <ResourceYaml
          ref="resourceyaml"
          :value="resource"
          :mode="mode"
          :yaml="resourceYaml"
          :offer-preview="mode==='edit'"
          :done-route="doneRoute"
          :done-override="resource.doneOverride"
        >
          <template #yamlFooter="{currentYaml, showPreview, yamlSave, yamlPreview, yamlUnpreview}">
            <div class="controls-row">
              <slot name="cru-yaml-footer">
                <div class="controls-right">
                  <button type="button" class="btn role-secondary" @click="checkCancel(true)">
                    <t k="generic.cancel" />
                  </button>
                </div>
                <div class="controls-middle">
                  <button
                    v-if="showPreview"
                    type="button"
                    class="btn role-secondary"
                    @click="yamlUnpreview"
                  >
                    <t k="resourceYaml.buttons.continue" />
                  </button>
                  <button
                    :disabled="resourceYaml === currentYaml"
                    type="button"
                    class="btn role-secondary"
                    @click="yamlPreview"
                  >
                    <t k="resourceYaml.buttons.diff" />
                  </button>
                </div>
                <div v-if="selectedSubtype || !subtypes.length" class="controls-right">
                  <button type="button" class="btn role-secondary" @click="checkCancel(false)">
                    <t k="generic.back" />
                  </button>
                  <button type="button" class="btn role-primary" @click="yamlSave(yamlCb)">
                    <t k="generic.create" />
                  </button>
                </div>
              </slot>
            </div>
          </template>
        </ResourceYaml>
      </section>

      <div v-for="(err, idx) in errors" :key="idx">
        <Banner color="error" :label="err" />
      </div>
    </form>

    <modal class="confirm-modal" name="cancel-modal" :width="400" height="auto">
      <div class="header">
        <h4 class="text-default-text">
          <t v-if="isCancelModal" k="generic.cancel" />
          <t v-else k="cruResource.backToForm" />
        </h4>
      </div>
      <div class="body">
        <p v-if="isCancelModal">
          <t k="cruResource.cancel" />
        </p>
        <p v-else>
          <t k="cruResource.back" />
        </p>
      </div>
      <div class="footer">
        <button type="button" class="btn role-secondary" @click="$modal.hide('cancel-modal')">
          <t k="cruResource.confirmYaml" />
        </button>
        <button type="button" class="btn role-primary" @click="confirmCancel(isCancelModal)">
          <span v-if="isCancelModal">
            <t k="cruResource.confirmCancel" />
          </span>
          <span v-else>
            <t k="cruResource.confirmBack" />
          </span>
        </button>
      </div>
    </modal>
  </section>
</template>

<style lang='scss'>
.cru-resource-yaml-container {
  .resource-yaml {
    .yaml-editor {
      min-height: 400px;
    }
  }
}
.confirm-modal {
  .v--modal-box {
    background-color: var(--default);
    box-shadow: none;
    min-height: 200px;
    .body {
      min-height: 75px;
      padding: 10px 0 0 15px;
      p {
        margin-top: 10px;
      }
    }
    .header {
      background-color: #4f3335;
      padding: 15px 0 0 15px;
    }
    .header,
    .footer {
      height: 50px;
    }
    .footer {
      border-top: 1px solid var(--border);
      text-align: center;
      padding: 10px 0 0 15px;
    }
  }
}
.subtypes-container {
  display: flex;
  flex-wrap: wrap;
  width: 100%;
}

.subtype-banner {
  border-left: 5px solid var(--primary);
  border-radius: var(--border-radius);
  display: flex;
  flex-basis: 40%;
  margin: 10px;
  min-height: 100px;
  padding: 10px;

  &.selected {
    background-color: var(--accent-btn);
  }

  &.top {
    background-image: linear-gradient(
      -90deg,
      var(--body-bg),
      var(--accent-btn)
    );

    H2 {
      margin: 0px;
    }

    .title {
      align-items: center;
      border-right: 1px solid var(--primary);
      display: flex;
      flex-basis: 10%;
      justify-content: space-evenly;
      margin-right: 20px;
      padding-right: 20px;
    }

    .description {
      color: var(--input-label);
      display: flex;
      flex-direction: column;
      justify-content: center;
    }
  }

  .description {
    color: var(--input-label);
  }

  &:not(.top) {
    align-items: center;
    box-shadow: 0px 0px 12px 3px var(--box-bg);
    flex-direction: row;
    justify-content: start;
    &:hover {
      cursor: pointer;
      box-shadow: 0px 0px 1px var(--outline-width) var(--outline);
    }
  }

  .round-image {
    background-color: var(--primary);
    border-radius: 50%;
    height: 50px;
    margin: 10px;
    min-width: 50px;
    overflow: hidden;
  }

  .banner-abbrv {
    align-items: center;
    background-color: var(--primary);
    color: white;
    display: flex;
    font-size: 2.5em;
    height: 100%;
    justify-content: center;
    width: 100%;
  }
}

.controls-row {
  margin-top: 20px;
  display: flex;
  justify-content: space-between;
}
</style>