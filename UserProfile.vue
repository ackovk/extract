<template>
  <div>
    <FormWrapper
      title="About you"
      body="Provide information about yourself"
    >
      <ValidationObserver v-slot="{ handleSubmit }">
        <form @submit.prevent="handleSubmit(updateProfileBasic)">
          <FormPhoto
            v-model="avatar"
            class="mb-4"
            :id="userId"
          />

          <FormGroup
            title="Email Address"
            disabled
            :value="email"
          />

          <ValidationProvider
            v-slot="{ errors }"
            name="First name"
            rules="required|name_with_specific_symbols"
          >
            <FormGroup
              v-model.trim="givenName"
              title="First Name*"
              placeholder="Enter first name"
              :errors="errors"
            />
          </ValidationProvider>

          <ValidationProvider
            v-slot="{ errors }"
            name="Last name"
            rules="required|name_with_specific_symbols"
          >
            <FormGroup
              v-model.trim="familyName"
              title="Last Name*"
              placeholder="Enter last name"
              :errors="errors"
            />
          </ValidationProvider>

          <ValidationProvider
            v-slot="{ errors }"
            name="Job title"
            rules="required|name_with_specific_symbols"
          >
            <FormGroup
              v-model.trim="jobTitle"
              title="Job title*"
              placeholder="Enter job title"
              :errors="errors"
            />
          </ValidationProvider>

          <ValidationProvider
            v-if="companyNameShown"
            v-slot="{ errors }"
            name="Company name"
            rules="max:160"
          >
            <FormGroup
              v-model.trim="companyName"
              title="Company"
              placeholder="Enter company name"
              :errors="errors"
            />
          </ValidationProvider>

          <ValidationProvider
            v-slot="{ errors }"
            name="Country"
            rules="required"
          >
            <FormGroup
              title="Country*"
              :errors="errors"
            >
              <ProfileCountrySelect v-model="country" />
            </FormGroup>
          </ValidationProvider>

          <div class="flex w-full justify-between">
            <ValidationProvider
              v-slot="{ errors }"
              name="State"
              :rules="stateValidationRules"
              class="mr-2 flex-1"
            >
              <RegionInput
                v-model="state"
                :countries="country"
                :errors="errors"
              />
            </ValidationProvider>

            <FormGroup
              v-model="postalCode"
              class="flex-1"
              name="postal_code"
              title="Postal / Zip Code"
              placeholder="Enter postal / zip code"
            />
          </div>

          <ValidationProvider
            v-slot="{ errors }"
            name="City"
            rules="required|name_with_specific_symbols"
          >
            <FormGroup
              v-model.trim="city"
              title="City*"
              placeholder="Enter city"
              :errors="errors"
            />
          </ValidationProvider>

          <ValidationProvider
            v-slot="{ errors }"
            name="Phone number"
            :rules="{phoneNumberLength:[phone.length]}"
          >
            <FormGroup
              v-if="phoneShown"
              title="Phone number"
              :errors="phoneTouched ? errors : []"
            >
              <MaskedPhoneInput
                v-model="phone"
                :invalid="Boolean(phoneTouched && errors.length)"
                @touched="phoneTouched = true"
              />
            </FormGroup>
          </ValidationProvider>

          <div class="flex justify-end mt-2 pt-3">
            <UiButton
              buttonText="Save"
              type="submit"
              :processing="isProcessing"
            />
          </div>
        </form>
      </ValidationObserver>
    </FormWrapper>
  </div>
</template>

<script lang="ts">
  import { Vue, Component, Prop, Watch } from "vue-property-decorator";

  import GET_PROFILE_BASIC from "@/apollo/profile/profileBasic/GetProfileBasic.gql";
  import UPDATE_USER_AVATAR from "@/apollo/profile/profileBasic/UpdateUserAvatar.gql";

  import FormPhoto from "@/components/form/FormPhoto.vue";
  import FormGroup from "@/components/form/FormGroup.vue";
  import FormWrapper from "@/components/form/FormWrapper.vue";
  import UiButton from '@/components/button/ui-button.vue';
  import FormInputGroup from "@/components/form/FormInputGroup.vue";
  import MaskedPhoneInput from "@/components/common/MaskedPhoneInput.vue";
  import ProfileCountrySelect from "@/components/autocompleteMultiselects/ProfileCountrySelect.vue";

  import { User } from "@/interface";
  import { NOTIFICATION } from "@/entities/Events";
  import Message from "@/data/notificationsMessages";
  import Roles from "@/entities/Roles";
  import lambdaRequest from "@/helpers/lambdaRequest";
  import { isEqual } from "lodash-es";
  import DropdownOption from "@/entities/DropdownOption";
  import RegionInput from "@/components/autocompleteMultiselects/RegionInput.vue";
  import getStateValidationRules from "@/helpers/locations/stateValidationRules";

  @Component({
    components: {
      RegionInput,
      FormPhoto,
      FormGroup,
      FormWrapper,
      UiButton,
      FormInputGroup,
      MaskedPhoneInput,
      ProfileCountrySelect,
    },
    apollo: {
      currentUser: {
        query: GET_PROFILE_BASIC,
        variables() {
          return {
            id: this.userId,
          };
        },
        update: (data) => {
          return data.user_by_pk;
        },
      },
    },
  })
  export default class ProfileBasic extends Vue {
    @Prop({ type: String, required: true }) readonly userId!: string;

    currentUser: null | User = null;

    phoneTouched: boolean = false;
    processing: boolean = false;

    email: string = "";
    avatar: string = "";
    givenName: string = "";
    familyName: string = "";
    jobTitle: string = "";
    country: DropdownOption[] = [];
    countryFromDB: string = "";
    state: DropdownOption[] = [];
    city: string = "";
    phone: string = "";
    companyName: string = "";
    postalCode: string = "";

    get isProcessing(): boolean {
      return this.processing || this.$apollo.queries.currentUser.loading;
    }

    @Watch("currentUser")
    watchCurrentUser(user: User): void {
      const country = user?.profile?.country;
      const state = user?.profile?.state;

      this.phone = user?.private_data?.phone
        ? String(user?.private_data?.phone)
        : "";
      this.email = user?.private_data?.email ?? "";
      this.avatar = user?.avatar ?? "";
      this.givenName = user?.given_name ?? "";
      this.familyName = user?.family_name ?? "";
      this.jobTitle = user?.profile?.job_title ?? "";
      this.companyName = user?.profile?.company_name ?? "";
      this.country = country ? [{id: country, name: country}] : [];
      this.state = state ? [{ id: state, name: state }] : [];
      this.city = user?.profile?.city ?? "";
      this.postalCode = user?.profile?.postal_code ?? "";

      this.countryFromDB = this.country[0]?.name;
    }

    @Watch("avatar")
    async watchAvatar(avatar: string): Promise<void> {
      if (avatar === this.currentUser?.avatar) {
        return;
      }

      this.updateAvatar(avatar);
    }

    @Watch("countryName", {immediate: false})
    clearCountry(newCountry: string): void {
      if (newCountry !== this.countryFromDB || !Boolean(newCountry)) {
        this.state = [];
        this.city = "";
        this.postalCode = "";
      }
    }

    get countryName(): string {
      return this.country[0]?.name || "";
    }

    get stateValidationRules(): string | undefined {
      return getStateValidationRules(this.countryName);
    }

    get phoneShown(): boolean {
      return Boolean(this.currentUser?.role !== Roles.ADMIN);
    }

    get companyNameShown(): boolean {
      return Boolean(this.currentUser?.role === Roles.CONTRACTOR);
    }

    async updateProfileBasic(): Promise<void> {
      if (this.isProcessing || !this.currentUser) {
        return;
      }

      this.processing = true;

      const {
        given_name,
        family_name,
        phone,
        profile,
      } = this.currentUser;

      const {
        city,
        country,
        state,
        job_title,
        company_name,
        postal_code
      } = profile;

      const oldUserData = {
        user: {
          given_name,
          family_name,
          phone
        },
        user_profile: {
          country,
          state,
          city,
          job_title,
          company_name,
          postal_code,
        },
      };

      const updatedUserData = {
        user: {
          given_name: this.givenName,
          family_name: this.familyName,
          phone: Number(this.phone) || undefined,
        },
        user_profile: {
          job_title: this.jobTitle,
          country: this.country[0].name,
          state: this.state[0]?.name,
          city: this.city,
          company_name: this.companyName,
          postal_code: this.postalCode,
        },
      };

      const userDataChanged = !isEqual(oldUserData, updatedUserData);

      try {
        if (userDataChanged) {
          await lambdaRequest({
            lambdaName: "updateUserPrimaryData",
            method: "put",
            requestData: updatedUserData,
          });

          this.refetch();
        }

        this.$eventBus.emit(
          NOTIFICATION,
          Message.successMessage("Your personal data has been successfully updated")
        );
      } catch (error) {
        this.$eventBus.emit(
          NOTIFICATION,
          Message.errorMessage(error.message)
        );
      } finally {
        this.processing = false;
      }
    }

    async updateAvatar(avatar: string): Promise<void> {
      const userData = {
        userId: this.userId,
        userSet: {
          avatar,
        },
      };

      try {
        await this.$apollo.mutate({
          mutation: UPDATE_USER_AVATAR,
          variables: userData,
        });

        this.$eventBus.emit(
          NOTIFICATION,
          Message.successMessage("Your avatar had been successfully updated")
        );
        this.refetch();
      } catch (e) {
        this.$eventBus.emit(
          NOTIFICATION,
          Message.errorMessage(e.message)
        );
      }
    }

    refetch(): void {
      this.$apollo.queries.currentUser.refetch();
      this.$emit("update");
    }
  }
</script>
