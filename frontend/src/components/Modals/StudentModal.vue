<template>
	<Dialog
		v-model="show"
		:options="{
			title: __('Add Students to Batch'),
			size: 'lg',
			actions: [
				{
					label: selectedStudents.length > 0 ? `Add ${selectedStudents.length} Students` : 'Add Students',
					variant: 'solid',
					onClick: (close) => addSelectedStudents(close),
					loading: studentResource.loading,
					disabled: selectedStudents.length === 0,
				},
			],
		}"
	>
		<template #body-content>
			<div class="flex flex-col gap-4">
				<!-- Search Input -->
				<FormControl
					type="text"
					v-model="searchQuery"
					:placeholder="__('Search users by name or email...')"
					class="w-full"
				/>

				<!-- Select All Toggle -->
				<div class="flex items-center justify-between border-b pb-2">
					<div class="flex items-center gap-2">
						<input
							type="checkbox"
							id="selectAll"
							v-model="selectAll"
							@change="toggleSelectAll"
							class="rounded border-gray-300"
						/>
						<label for="selectAll" class="text-sm font-medium cursor-pointer">
							{{ __('Select All') }} ({{ filteredUsers.length }} users)
						</label>
					</div>
					<div class="text-sm text-gray-600">
						{{ selectedStudents.length }} {{ __('selected') }}
					</div>
				</div>

				<!-- User List -->
				<div class="max-h-96 overflow-y-auto border rounded">
					<div v-if="usersResource.loading" class="text-center py-8">
						<div class="animate-spin rounded-full h-8 w-8 border-b-2 border-blue-600 mx-auto"></div>
						<p class="mt-2 text-gray-600">{{ __('Loading users...') }}</p>
					</div>
					<div v-else-if="filteredUsers.length === 0" class="text-center py-8 text-gray-500">
						<p>{{ searchQuery ? __('No users found') : __('No available users') }}</p>
						<p class="text-sm mt-1">{{ __('All users may already be enrolled in this batch') }}</p>
					</div>
					<div v-else class="divide-y">
						<div
							v-for="user in filteredUsers"
							:key="user.name"
							class="flex items-center gap-3 p-3 hover:bg-gray-50 cursor-pointer"
							@click="toggleUserSelection(user.name)"
						>
							<input
								type="checkbox"
								:id="user.name"
								v-model="selectedStudents"
								:value="user.name"
								class="rounded border-gray-300"
								@click.stop
							/>
							<Avatar
								:image="user.user_image"
								:label="user.full_name || user.name"
								size="sm"
							/>
							<div class="flex-1">
								<div class="font-medium">{{ user.full_name || user.name }}</div>
								<div class="text-sm text-gray-600">{{ user.email }}</div>
							</div>
						</div>
					</div>
				</div>

				<!-- Selected Count Summary -->
				<div v-if="selectedStudents.length > 0" class="bg-blue-50 p-3 rounded border">
					<p class="text-sm text-blue-800">
						{{ selectedStudents.length }} {{ selectedStudents.length === 1 ? 'student' : 'students' }} selected for enrollment
					</p>
				</div>
			</div>
		</template>
	</Dialog>
</template>
<script setup>
import { Dialog, createResource, toast, FormControl, Avatar } from 'frappe-ui'
import { ref, inject, computed, watch } from 'vue'
import { useOnboarding } from 'frappe-ui/frappe'

const students = defineModel('reloadStudents')
const batchModal = defineModel('batchModal')
const user = inject('$user')
const { updateOnboardingStep } = useOnboarding('learning')
const show = defineModel()

// Reactive variables
const searchQuery = ref('')
const selectedStudents = ref([])
const selectAll = ref(false)

const props = defineProps({
	batch: {
		type: String,
		default: null,
	},
})

// Get all users who are not already in the batch
const usersResource = createResource({
	url: 'lms.lms.api.get_users_not_in_batch',
	params: {
		batch: props.batch,
	},
	auto: true,
})

// Filtered users based on search query
const filteredUsers = computed(() => {
	if (!usersResource.data) return []
	
	let users = usersResource.data
	
	if (searchQuery.value) {
		const query = searchQuery.value.toLowerCase()
		users = users.filter(user => 
			(user.full_name && user.full_name.toLowerCase().includes(query)) ||
			(user.email && user.email.toLowerCase().includes(query)) ||
			(user.name && user.name.toLowerCase().includes(query))
		)
	}
	
	return users
})

// Toggle individual user selection
const toggleUserSelection = (userId) => {
	const index = selectedStudents.value.indexOf(userId)
	if (index > -1) {
		selectedStudents.value.splice(index, 1)
	} else {
		selectedStudents.value.push(userId)
	}
}

// Toggle select all
const toggleSelectAll = () => {
	if (selectAll.value) {
		selectedStudents.value = filteredUsers.value.map(user => user.name)
	} else {
		selectedStudents.value = []
	}
}

// Watch for changes in selected students to update select all checkbox
watch(selectedStudents, () => {
	selectAll.value = selectedStudents.value.length === filteredUsers.value.length && filteredUsers.value.length > 0
}, { deep: true })

// Watch for changes in filtered users to update select all state
watch(filteredUsers, () => {
	if (filteredUsers.value.length === 0) {
		selectAll.value = false
	} else {
		selectAll.value = selectedStudents.value.length === filteredUsers.value.length
	}
})

// Resource for bulk adding students
const studentResource = createResource({
	url: 'lms.lms.api.bulk_add_batch_students',
	makeParams() {
		return {
			batch: props.batch,
			students: selectedStudents.value,
		}
	},
})

const addSelectedStudents = (close) => {
	if (selectedStudents.value.length === 0) {
		toast.error(__('Please select at least one student'))
		return
	}

	studentResource.submit(
		{},
		{
			onSuccess(data) {
				if (user.data?.is_system_manager)
					updateOnboardingStep('add_batch_student')

				students.value.reload()
				batchModal.value.reload()
				
				toast.success(__(`Successfully added ${selectedStudents.value.length} student(s) to the batch`))
				
				// Reset selections
				selectedStudents.value = []
				selectAll.value = false
				searchQuery.value = ''
				
				close()
			},
			onError(err) {
				toast.error(err.messages?.[0] || err.message || __('Failed to add students'))
			},
		}
	)
}

// Reset when modal opens
watch(show, (newValue) => {
	if (newValue) {
		selectedStudents.value = []
		selectAll.value = false
		searchQuery.value = ''
		usersResource.reload()
	}
})
</script>