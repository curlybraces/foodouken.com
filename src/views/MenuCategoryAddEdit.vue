<template>
  <div class="w-full">
    <ImageUpload v-model="images" />
    <hr class="border-white border-opacity-12 my-8" />

    <div class="mx-4">
      <span class="tg-h3-mobile text-white text-opacity-54 my-6 inline-block">
        Category info
      </span>
      <MaterialInput
        v-model="$v.category.name.$model"
        label="Category name"
        labelBg="bg-background"
        :error="$v.category.name.$error"
      >
        <span v-if="$v.category.name.$error" class="text-red-600 text-xs">
          Name is required
        </span>
      </MaterialInput>
      <TextArea
        v-model="category.description"
        label="Description"
        labelBg="bg-background"
      />
    </div>
    <hr class="border-white border-opacity-12 my-8" />

    <div class="my-8 mx-4 text-sm text-red-600">
      <span v-if="$v.$invalid && submitError">
        Please fill out the form properly.
      </span>
      <BaseAPIErrorDisplay :error="apiError" />
    </div>

    <div class="px-4 mb-8 max-w-sm mx-auto">
      <Button
        title="Save category"
        class="rounded-full"
        padding="p-2"
        @clicked="submit"
      />
    </div>
  </div>
</template>

<script>
import ImageUpload from '@/components/imageUpload/ImageUpload.vue';
import MaterialInput from '@/components/inputs/MaterialInput';
import TextArea from '@/components/inputs/TextArea';
import Button from '@/components/Button';
import BaseAPIErrorDisplay from '@/components/BaseAPIErrorDisplay';

import { mapActions } from 'vuex';
import { required } from 'vuelidate/lib/validators';
import { sleep } from '@/helpers.js';

export default {
  name: 'MenuCategoryAddEdit',
  components: {
    ImageUpload,
    MaterialInput,
    TextArea,
    Button,
    BaseAPIErrorDisplay
  },
  validations: {
    category: {
      name: {
        required
      }
    }
  },
  data() {
    return {
      edit: false,
      params: {
        tenantSlug: this.$route.params.tenantSlug,
        categoryId: this.$route.params.categoryId
      },
      submitError: false,
      apiError: null,
      category: {
        name: '',
        description: '',
        imageUrl:
          'https://res.cloudinary.com/whynotearth/image/upload/v1593327134/foodouken/tenant_upload/b6pit9hqniikb1jnz5px.png'
      }
    };
  },
  created() {
    this.init();
  },
  beforeDestroy() {
    this.category = {};
    this.submitError = false;
    this.apiError = '';
  },
  computed: {
    images: {
      //FIXME: ImageUpload component should handle strings, the solution below is a temporary fix.
      get() {
        return [{ secure_url: this.category.imageUrl }];
      },
      set(value) {
        this.category.imageUrl = value[0] ? value[0].secure_url : '';
      }
    }
  },
  methods: {
    ...mapActions('auth', ['ping']),
    ...mapActions('menu', [
      'fetchTenantCategoryById',
      'createTenantCategory',
      'updateTenantCategory'
    ]),
    init() {
      this.edit = this.params.categoryId !== undefined ? true : false;
      if (this.edit) {
        this.fetchTenantCategoryById(this.params)
          .then(category => {
            this.category = category;
          })
          .catch(error => {
            this.apiError = error.response.data
              ? error.response.data
              : 'Failed to fetch category details, please refresh.';
            throw error;
          });
      }
    },
    async onSuccessSubmit() {
      this.$store.commit('overlay/updateModel', {
        title: 'Success!',
        message: 'Your category has been saved!'
      });

      await sleep(1000);

      await this.$router.push({
        name: 'MenuCategoryList'
      });

      this.$store.commit('overlay/updateModel', {
        title: '',
        message: ''
      });
    },
    submit() {
      this.$v.$touch();
      if (this.$v.$invalid) {
        this.submitError = true;
        return false;
      }
      this.ping()
        .then(user => {
          if (user.isAuthenticated) {
            let payload = {
              tenantSlug: this.params.tenantSlug,
              category: this.category
            };
            this.edit ? this.editItem(payload) : this.newItem(payload);
          }
        })
        .catch(error => {
          this.$router.push({ name: 'Welcome' });
          throw error;
        });
    },
    newItem(payload) {
      this.createTenantCategory(payload)
        .then(() => {
          this.onSuccessSubmit();
        })
        .catch(error => {
          this.apiError = error.response.data
            ? error.response.data
            : 'Something went wrong, try again.';
          throw error;
        });
    },
    editItem(payload) {
      payload.categoryId = this.params.categoryId;
      this.updateTenantCategory(payload)
        .then(() => {
          this.onSuccessSubmit();
        })
        .catch(error => {
          this.apiError = error.response.data
            ? error.response.data
            : 'Something went wrong, try again.';
          throw error;
        });
    }
  }
};
</script>
