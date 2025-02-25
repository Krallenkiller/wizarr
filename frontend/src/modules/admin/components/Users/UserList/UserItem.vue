<template>
    <ListItem>
        <template #icon>
            <div class="flex-shrink-0 h-[60px] w-[60px] rounded bg-gray-50 overflow-hidden">
                <img :src="profilePicture" :onerror="`this.src='${backupPicture}'`" class="w-full h-full object-cover object-center" alt="Profile Picture" />
            </div>
        </template>
        <template #title>
            <span class="text-lg">{{ user.username }}</span>
            <div class="flex flex-col">
                <p v-if="user.email" class="text-xs truncate text-gray-500 dark:text-gray-400 w-full">{{ user.email }}</p>
                <p v-else class="text-xs truncate text-gray-500 dark:text-gray-400 w-full">No email</p>
                <p v-if="user.expires" class="text-xs truncate w-full" :class="color">{{ expired }}</p>
                <p v-else class="text-xs truncate text-gray-500 dark:text-gray-400 w-full">{{ $filter("timeAgo", user.created) }}</p>
            </div>
        </template>
        <template #buttons>
            <div class="flex flex-row space-x-2">
                <div v-if="user.code" class="border border-gray-200 dark:border-gray-700 rounded p-2 text-xs text-gray-500 dark:text-gray-400">
                    <span>{{ user.code }}</span>
                </div>
                <FormKit type="button" data-theme="secondary" @click="viewUser" :classes="{ input: '!bg-secondary !px-3.5 h-[36px]' }">
                    <i class="fa-solid fa-eye"></i>
                </FormKit>
            </div>
            <div class="flex flex-row space-x-2">
                <FormKit type="button" data-theme="secondary" @click="manageUser" :classes="{ input: '!bg-secondary !px-3.5 h-[36px]' }">
                    <i class="fa-solid fa-edit"></i>
                </FormKit>
                <FormKit type="button" data-theme="danger" :disabled="disabled.delete" @click="localDeleteUser" :classes="{ input: '!bg-red-600 !px-3.5 h-[36px]' }">
                    <i class="fa-solid fa-trash"></i>
                </FormKit>
            </div>
        </template>
    </ListItem>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import { mapActions, mapState } from "pinia";
import { useUsersStore } from "@/stores/users";
import { useServerStore } from "@/stores/server";

import type { User } from "@/types/api/users";
import type { CustomModalOptions } from "@/plugins/modal";

import ListItem from "@/components/ListItem.vue";
import UserManager from "../UserManager/UserManager.vue";

export default defineComponent({
    name: "UserItem",
    components: {
        ListItem,
    },
    props: {
        user: {
            type: Object as () => User,
            required: true,
        },
    },
    data() {
        return {
            profilePicture: "https://ui-avatars.com/api/?uppercase=true&name=" + this.user.username + "&length=1",
            backupPicture: "https://ui-avatars.com/api/?uppercase=true&name=" + this.user.username + "&length=1",
            disabled: {
                delete: false,
            },
        };
    },
    computed: {
        expired(): string {
            if (this.$filter("isPast", this.user.expires!)) {
                return this.__("Expired %{s}", { s: this.$filter("timeAgo", this.user.expires!) });
            } else {
                return this.__("Expires %{s}", { s: this.$filter("timeAgo", this.user.expires!) });
            }
        },
        color() {
            const inHalfDay = new Date();
            inHalfDay.setHours(inHalfDay.getHours() + 12);

            if (this.$filter("isPast", this.user.expires!)) {
                return "text-red-600 dark:text-red-500";
            }

            if (this.$filter("dateLess", this.user.expires!, inHalfDay)) {
                return "text-yellow-500 dark:text-yellow-400";
            }

            return "text-gray-500 dark:text-gray-400";
        },
        ...mapState(useServerStore, ["settings"]),
    },
    methods: {
        async getProfilePicture() {
            const response = this.$axios.get(`/api/users/${this.user.token}/profile-picture`, {
                responseType: "blob",
            });

            this.profilePicture = URL.createObjectURL((await response).data);
        },
        async manageUser() {
            const modal_options: CustomModalOptions = {
                title: this.__(`Managing %{user}`, { user: this.user.username }),
                buttons: [
                    {
                        text: this.__("Save"),
                        attrs: {
                            "data-theme": "primary",
                            disabled: true,
                        },
                        emit: "saveUser",
                    },
                ],
            };

            const modal_props = {
                user: this.user,
            };

            this.$modal.openModal(UserManager, modal_options, modal_props);
        },
        async viewUser() {
            // Switch statement to determine which server type is being used and its respective URL
            switch (this.settings.server_type) {
                case "jellyfin":
                    window.open(`${this.settings.server_url}/web/index.html#!/useredit.html?userId=${this.user.token}`, "_blank");
                    break;
                case "plex":
                    window.open(`${this.settings.server_url}/web/index.html#!/settings/manage-library-access/sharing/${this.user.token}`, "_blank");
                    break;
            }
        },
        async localDeleteUser() {
            // Confirm the user wants to delete the user
            if (await this.$modal.confirmModal(this.__("Are you sure?"), this.__("Do you really want to delete this user from your media server?"))) {
                this.disabled.delete = true;
                await this.deleteUser(this.user.id).finally(() => (this.disabled.delete = false));
                this.$toast.info(this.__(`User %{user} deleted`, { user: this.user.username }));
            } else {
                this.$toast.info(this.__("User deletion cancelled"));
            }
        },
        ...mapActions(useUsersStore, ["deleteUser"]),
    },
    mounted() {
        this.getProfilePicture();
    },
});
</script>
