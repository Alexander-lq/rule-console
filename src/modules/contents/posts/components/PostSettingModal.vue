<script lang="ts" setup>
import { Toast, VButton, VModal, VSpace } from "@halo-dev/components";
import { computed, nextTick, ref, watchEffect } from "vue";
import type { Post } from "@halo-dev/api-client";
import cloneDeep from "lodash.clonedeep";
import { apiClient } from "@/utils/api-client";
import { useThemeCustomTemplates } from "@/modules/interface/themes/composables/use-theme";
import { postLabels } from "@/constants/labels";
import { randomUUID } from "@/utils/id";
import { toDatetimeLocal, toISOString } from "@/utils/date";
import AnnotationsForm from "@/components/form/AnnotationsForm.vue";
import { submitForm } from "@formkit/core";

const initialFormState: Post = {
  spec: {
    title: "",
    slug: "",
    template: "",
    cover: "",
    deleted: false,
    publish: false,
    publishTime: "",
    pinned: false,
    allowComment: true,
    visible: "PUBLIC",
    priority: 0,
    excerpt: {
      autoGenerate: true,
      raw: "",
    },
    categories: [],
    tags: [],
    htmlMetas: [],
  },
  apiVersion: "content.halo.run/v1alpha1",
  kind: "Post",
  metadata: {
    name: randomUUID(),
  },
};

const props = withDefaults(
  defineProps<{
    visible: boolean;
    post?: Post;
    publishSupport?: boolean;
    onlyEmit?: boolean;
  }>(),
  {
    visible: false,
    post: undefined,
    publishSupport: true,
    onlyEmit: false,
  }
);

const emit = defineEmits<{
  (event: "update:visible", visible: boolean): void;
  (event: "close"): void;
  (event: "saved", post: Post): void;
  (event: "published", post: Post): void;
}>();

const formState = ref<Post>(cloneDeep(initialFormState));
const saving = ref(false);
const publishing = ref(false);
const publishCanceling = ref(false);
const submitType = ref<"publish" | "save">();

const isUpdateMode = computed(() => {
  return !!formState.value.metadata.creationTimestamp;
});

const handleVisibleChange = (visible: boolean) => {
  emit("update:visible", visible);
  if (!visible) {
    emit("close");
  }
};

const handleSubmit = () => {
  if (submitType.value === "publish") {
    handlePublish();
  }
  if (submitType.value === "save") {
    handleSave();
  }
};

const handleSaveClick = () => {
  submitType.value = "save";

  nextTick(() => {
    submitForm("post-setting-form");
  });
};

const handlePublishClick = () => {
  if (!props.onlyEmit) {
    handlePublish();
  }

  submitType.value = "publish";

  nextTick(() => {
    submitForm("post-setting-form");
  });
};

const handleSave = async () => {
  annotationsFormRef.value?.handleSubmit();
  await nextTick();

  const { customAnnotations, annotations, customFormInvalid, specFormInvalid } =
    annotationsFormRef.value || {};

  if (customFormInvalid || specFormInvalid) {
    return;
  }

  formState.value.metadata.annotations = {
    ...annotations,
    ...customAnnotations,
  };

  if (props.onlyEmit) {
    emit("saved", formState.value);
    return;
  }

  try {
    saving.value = true;

    const { data } = isUpdateMode.value
      ? await apiClient.extension.post.updatecontentHaloRunV1alpha1Post({
          name: formState.value.metadata.name,
          post: formState.value,
        })
      : await apiClient.extension.post.createcontentHaloRunV1alpha1Post({
          post: formState.value,
        });

    formState.value = data;
    emit("saved", data);

    handleVisibleChange(false);

    Toast.success("????????????");
  } catch (e) {
    console.error("Failed to save post", e);
  } finally {
    saving.value = false;
  }
};

const handlePublish = async () => {
  if (props.onlyEmit) {
    emit("published", formState.value);
    return;
  }

  try {
    publishing.value = true;

    const { data } = await apiClient.post.publishPost({
      name: formState.value.metadata.name,
    });

    formState.value = data;

    emit("published", data);

    handleVisibleChange(false);

    Toast.success("????????????");
  } catch (e) {
    console.error("Failed to publish post", e);
  } finally {
    publishing.value = false;
  }
};

const handleUnpublish = async () => {
  try {
    publishCanceling.value = true;

    await apiClient.post.unpublishPost({
      name: formState.value.metadata.name,
    });

    handleVisibleChange(false);

    Toast.success("??????????????????");
  } catch (e) {
    console.error("Failed to publish post", e);
  } finally {
    publishCanceling.value = false;
  }
};

watchEffect(() => {
  if (props.post) {
    formState.value = cloneDeep(props.post);
  }
});

// custom templates
const { templates } = useThemeCustomTemplates("post");

// publishTime convert
const publishTime = computed(() => {
  const { publishTime } = formState.value.spec;
  if (publishTime) {
    console.log(toDatetimeLocal(publishTime));
    return toDatetimeLocal(publishTime);
  }
  return "";
});

const onPublishTimeChange = (value: string) => {
  formState.value.spec.publishTime = value ? toISOString(value) : undefined;
};

const annotationsFormRef = ref<InstanceType<typeof AnnotationsForm>>();
</script>
<template>
  <VModal
    :visible="visible"
    :width="700"
    title="????????????"
    :centered="false"
    @update:visible="handleVisibleChange"
  >
    <template #actions>
      <slot name="actions"></slot>
    </template>

    <FormKit
      id="post-setting-form"
      type="form"
      name="post-setting-form"
      :config="{ validationVisibility: 'submit' }"
      @submit="handleSubmit"
    >
      <div>
        <div class="md:grid md:grid-cols-4 md:gap-6">
          <div class="md:col-span-1">
            <div class="sticky top-0">
              <span class="text-base font-medium text-gray-900">
                ????????????
              </span>
            </div>
          </div>
          <div class="mt-5 divide-y divide-gray-100 md:col-span-3 md:mt-0">
            <FormKit
              v-model="formState.spec.title"
              label="??????"
              type="text"
              name="title"
              validation="required|length:0,100"
            ></FormKit>
            <FormKit
              v-model="formState.spec.slug"
              label="??????"
              name="slug"
              type="text"
              validation="required|length:0,100"
            ></FormKit>
            <FormKit
              v-model="formState.spec.categories"
              label="????????????"
              name="categories"
              type="categoryCheckbox"
            />
            <FormKit
              v-model="formState.spec.tags"
              label="??????"
              name="tags"
              type="tagCheckbox"
            />
            <FormKit
              v-model="formState.spec.excerpt.autoGenerate"
              :options="[
                { label: '???', value: true },
                { label: '???', value: false },
              ]"
              name="autoGenerate"
              label="??????????????????"
              type="radio"
            >
            </FormKit>
            <FormKit
              v-if="!formState.spec.excerpt.autoGenerate"
              v-model="formState.spec.excerpt.raw"
              label="???????????????"
              name="raw"
              type="textarea"
              :rows="5"
              validation="length:0,1024"
            ></FormKit>
          </div>
        </div>

        <div class="py-5">
          <div class="border-t border-gray-200"></div>
        </div>

        <div class="md:grid md:grid-cols-4 md:gap-6">
          <div class="md:col-span-1">
            <div class="sticky top-0">
              <span class="text-base font-medium text-gray-900">
                ????????????
              </span>
            </div>
          </div>
          <div class="mt-5 divide-y divide-gray-100 md:col-span-3 md:mt-0">
            <FormKit
              v-model="formState.spec.allowComment"
              :options="[
                { label: '???', value: true },
                { label: '???', value: false },
              ]"
              label="????????????"
              type="radio"
            ></FormKit>
            <FormKit
              v-model="formState.spec.pinned"
              :options="[
                { label: '???', value: true },
                { label: '???', value: false },
              ]"
              label="????????????"
              name="pinned"
              type="radio"
            ></FormKit>
            <FormKit
              v-model="formState.spec.visible"
              :options="[
                { label: '??????', value: 'PUBLIC' },
                { label: '??????', value: 'PRIVATE' },
              ]"
              label="?????????"
              name="visible"
              type="select"
            ></FormKit>
            <FormKit
              :model-value="publishTime"
              label="????????????"
              type="datetime-local"
              @input="onPublishTimeChange"
            ></FormKit>
            <FormKit
              v-model="formState.spec.template"
              :options="templates"
              label="???????????????"
              name="template"
              type="select"
            ></FormKit>
            <FormKit
              v-model="formState.spec.cover"
              name="cover"
              label="?????????"
              type="attachment"
              validation="length:0,1024"
            ></FormKit>
          </div>
        </div>
      </div>
    </FormKit>

    <div class="py-5">
      <div class="border-t border-gray-200"></div>
    </div>

    <div class="md:grid md:grid-cols-4 md:gap-6">
      <div class="md:col-span-1">
        <div class="sticky top-0">
          <span class="text-base font-medium text-gray-900"> ????????? </span>
        </div>
      </div>
      <div class="mt-5 divide-y divide-gray-100 md:col-span-3 md:mt-0">
        <AnnotationsForm
          :key="formState.metadata.name"
          ref="annotationsFormRef"
          :value="formState.metadata.annotations"
          kind="Post"
          group="content.halo.run"
        />
      </div>
    </div>

    <template #footer>
      <VSpace>
        <template v-if="publishSupport">
          <VButton
            v-if="formState.metadata.labels?.[postLabels.PUBLISHED] !== 'true'"
            :loading="publishing"
            type="secondary"
            @click="handlePublishClick()"
          >
            ??????
          </VButton>
          <VButton
            v-else
            :loading="publishCanceling"
            type="danger"
            @click="handleUnpublish()"
          >
            ????????????
          </VButton>
        </template>
        <VButton :loading="saving" type="secondary" @click="handleSaveClick()">
          ??????
        </VButton>
        <VButton type="default" @click="handleVisibleChange(false)">
          ??????
        </VButton>
      </VSpace>
    </template>
  </VModal>
</template>
