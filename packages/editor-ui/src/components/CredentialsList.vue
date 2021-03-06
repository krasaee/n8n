<template>
	<div v-if="dialogVisible">
		<credentials-edit :dialogVisible="credentialEditDialogVisible" @closeDialog="closeCredentialEditDialog" @credentialsUpdated="reloadCredentialList" @credentialsCreated="reloadCredentialList" :setCredentialType="editCredentials && editCredentials.type" :editCredentials="editCredentials"></credentials-edit>

		<el-dialog :visible="dialogVisible" append-to-body width="80%" title="Credentials" :before-close="closeDialog">
			<div class="text-very-light">
				Your saved credentials:
			</div>

			<el-button title="Create New Credentials" class="new-credentials-button" @click="createCredential()">
				<font-awesome-icon icon="plus" />
				<div class="next-icon-text">
					Add New
				</div>
			</el-button>

			<el-table :data="credentials" :default-sort = "{prop: 'name', order: 'ascending'}" stripe @row-click="editCredential" max-height="450" v-loading="isDataLoading">
				<el-table-column property="name" label="Name" class-name="clickable" sortable></el-table-column>
				<el-table-column property="type" label="Type" class-name="clickable" sortable>
					<template slot-scope="scope">
						{{credentialTypeDisplayNames[scope.row.type]}}
					</template>
				</el-table-column>
				<el-table-column property="createdAt" label="Created" class-name="clickable" sortable></el-table-column>
				<el-table-column property="updatedAt" label="Updated" class-name="clickable" sortable></el-table-column>
				<el-table-column
					label="Operations"
					width="180">
					<template slot-scope="scope">
						<el-button title="Edit Credentials" @click.stop="editCredential(scope.row)" icon="el-icon-edit" circle></el-button>
						<el-button title="Delete Credentials" @click.stop="deleteCredential(scope.row)" type="danger" icon="el-icon-delete" circle></el-button>
						<!-- Would be nice to have this button switch from connect to disconnect based on the credential status -->
						<el-button title="Connect OAuth Credentials" @click.stop="OAuth2CredentialAuthorize(scope.row)" icon="el-icon-caret-right" v-if="scope.row.type == 'OAuth2Api'" circle></el-button>
					</template>
				</el-table-column>
			</el-table>
		</el-dialog>
	</div>
</template>

<script lang="ts">
import { restApi } from '@/components/mixins/restApi';
import { ICredentialsResponse } from '@/Interface';
import { nodeHelpers } from '@/components/mixins/nodeHelpers';
import { showMessage } from '@/components/mixins/showMessage';
import CredentialsEdit from '@/components/CredentialsEdit.vue';
import { genericHelpers } from '@/components/mixins/genericHelpers';

import mixins from 'vue-typed-mixins';

export default mixins(
	genericHelpers,
	nodeHelpers,
	restApi,
	showMessage,
).extend({
	name: 'CredentialsList',
	props: [
		'dialogVisible',
	],
	components: {
		CredentialsEdit,
	},
	data () {
		return {
			credentialEditDialogVisible: false,
			credentialTypeDisplayNames: {} as { [key: string]: string; },
			credentials: [] as ICredentialsResponse[],
			displayAddCredentials: false,
			editCredentials: null as ICredentialsResponse | null,
			isDataLoading: false,
		};
	},
	watch: {
		dialogVisible (newValue) {
			if (newValue) {
				this.loadCredentials();
				this.loadCredentialTypes();
			}
		},
	},
	methods: {
		closeCredentialEditDialog () {
			this.credentialEditDialogVisible = false;
		},
		closeDialog () {
			// Handle the close externally as the visible parameter is an external prop
			// and is so not allowed to be changed here.
			this.$emit('closeDialog');
			return false;
		},
		createCredential () {
			this.editCredentials = null;
			this.credentialEditDialogVisible = true;
		},
		async OAuth2CredentialAuthorize (credential: ICredentialsResponse) {
			let url;
			try {
				url = await this.restApi().OAuth2CredentialAuthorize(credential) as string;
			} catch (error) {
				this.$showError(error, 'OAuth Authorization Error', 'Error generating authorization URL:');
				return;
			}

			const params = `scrollbars=no,resizable=no,status=no,location=no,toolbar=no,menubar=no,width=0,height=0,left=-1000,top=-1000`;
			const oauthPopup = window.open(url, 'OAuth2 Authorization', params);

			console.log(oauthPopup);
		},
		editCredential (credential: ICredentialsResponse) {
			const editCredentials = {
				id: credential.id,
				name: credential.name,
				type: credential.type,
			} as ICredentialsResponse;

			this.editCredentials = editCredentials;
			this.credentialEditDialogVisible = true;
		},
		reloadCredentialList () {
			this.loadCredentials();
		},
		loadCredentialTypes () {
			if (Object.keys(this.credentialTypeDisplayNames).length !== 0) {
				// Data is already loaded
				return;
			}

			if (this.$store.getters.allCredentialTypes === null) {
				// Data is not ready yet to be loaded
				return;
			}

			for (const credentialType of this.$store.getters.allCredentialTypes) {
				this.credentialTypeDisplayNames[credentialType.name] = credentialType.displayName;
			}
		},
		loadCredentials () {
			this.isDataLoading = true;
			try {
				this.credentials = JSON.parse(JSON.stringify(this.$store.getters.allCredentials));
			} catch (error) {
				this.$showError(error, 'Problem loading credentials', 'There was a problem loading the credentials:');
				this.isDataLoading = false;
				return;
			}

			this.credentials.forEach((credentialData: ICredentialsResponse) => {
				credentialData.createdAt = this.convertToDisplayDate(credentialData.createdAt as number);
				credentialData.updatedAt = this.convertToDisplayDate(credentialData.updatedAt as number);
			});

			this.isDataLoading = false;
		},

		async deleteCredential (credential: ICredentialsResponse) {
			const deleteConfirmed = await this.confirmMessage(`Are you sure that you want to delete the credentials "${credential.name}"?`, 'Delete Credentials?', 'warning', 'Yes, delete!');

			if (deleteConfirmed === false) {
				return;
			}

			try {
				await this.restApi().deleteCredentials(credential.id!);
			} catch (error) {
				this.$showError(error, 'Problem deleting credentials', 'There was a problem deleting the credentials:');
				return;
			}

			// Remove also from local store
			this.$store.commit('removeCredentials', credential);

			// Now that the credentials got removed check if any nodes used them
			this.updateNodesCredentialsIssues();

			this.$showMessage({
				title: 'Credentials deleted',
				message: `The credential "${credential.name}" got deleted!`,
				type: 'success',
			});

			// Refresh list
			this.loadCredentials();
		},
	},
});
</script>

<style lang="scss">

.new-credentials-button {
	float: right;
	position: relative;
	top: -15px;
}

</style>
