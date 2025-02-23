<script setup>
import { ref, watch, onMounted } from 'vue';
import { getData, getDataById, deleteData, postData, updateData } from '@/lib/fetchMethod.js';
import { TaskManagement } from '@/lib/TaskManagement.js'
import { useRoute } from 'vue-router';
import router from '@/router';
import Toast from '@/components/Toast.vue'
import AddPopup from '@/components/AddPopup.vue';
import EditPopup from '@/components/EditPopup.vue'
import DeletePopup from '@/components/DeletePopup.vue'
import TaskDetail from '@/components/TaskDetail.vue';
import { styleStatus } from '@/lib/styleStatus';

const route = useRoute()

const taskManagement = ref(new TaskManagement())

const fetch = async() => {
    // Fetch data
    const result = await getData(import.meta.env.VITE_URL)
    // Front-end
    taskManagement.value.addAllTask(result)
}

const items = taskManagement.value.getAllTask()
watch(items, (newItems) => {
    newItems.forEach(item => {
        switch(item.status) {
            case 'TO_DO':
                item.status = 'To Do'
                break;
            case 'NO_STATUS':
                item.status = 'No Status'
                break;
            case 'DOING':
                item.status = 'Doing'
                break;
            case 'DONE':
                item.status = 'Done'
                break;
            default:
                break;
        }
    })
})

onMounted(async () => {
    await fetch()
    const detailId = route.params.detailId
    const editId = route.params.editId
    if(detailId) {
        await openTaskDetail(detailId)
    } else if (editId) {
        await openEditPopup(editId)
    }
})

// Toast
const toastHandle = ref()

//Task Detail
const taskPopupStatus = ref(false);
const taskDetailTarget = ref()
const openTaskDetail = async (componentId) => {
    router.push({ name: 'TaskDetail', params: { detailId: componentId } })
    taskDetailTarget.value = ''
    taskPopupStatus.value = true
    const result =  await getDataById(import.meta.env.VITE_URL, componentId)
    result.createdOn = new Date(result.createdOn).toLocaleString('en-AU', options)
    result.updatedOn = new Date(result.updatedOn).toLocaleString('en-AU', options)
    taskDetailTarget.value = result
}
const closeTaskDetail = () => {
    taskPopupStatus.value = false
    router.push('/task')
}

// Edit
const options = {
  timeZoneName: "short",
  hour12: false,
}
const targetItem = ref()
const editPopupStatus = ref(false)
const openEditPopup = async (componentId) => {
    router.push({ name: 'EditPopup', params: { editId: componentId }})
    editPopupStatus.value = true
    const result = await getDataById(import.meta.env.VITE_URL, componentId)
    result.createdOn = new Date(result.createdOn).toLocaleString('en-AU', options)
    result.updatedOn = new Date(result.updatedOn).toLocaleString('en-AU', options)
    targetItem.value = result
}
const closeEditPopup = () => {
    targetItem.value = ''
    editPopupStatus.value = false
    router.push('/task')
}
const updateEdit = async (newEdit) => {
    const response = await updateData(import.meta.env.VITE_URL,
    {
        title: newEdit.title,
        description: newEdit.description,
        assignees: newEdit.assignees,
        status: newEdit.status
    }, newEdit.id)
    if(response.ok) {
        // Toast
        toastHandle.value = {type: 'success', status: true, message: `Task successfully edited to ${newEdit.title} !`}
        // Front-end
        taskManagement.value.updateTask(newEdit, newEdit.id)
        editPopupStatus.value = false
        router.push('/task')
    }
}

// Add
const addPopupStatus = ref(false)
const openAddPopup = () => {
  addPopupStatus.value = true
}
const closeAddPopup = () => {
  addPopupStatus.value = false
  router.push('/task')
}
const confirmAdd = async (newTask) => {
    // Back-end
    const response = await postData(import.meta.env.VITE_URL, newTask)
    if(response.ok) {
        // Add Toast
        toastHandle.value = {type: 'success', status: true, message: `${newTask.title} task added successfully!`}
        // Front-end
        taskManagement.value.addTask(await response.json())
        addPopupStatus.value = false
    }
}

// Delete
const deletePopupStatus = ref(false)
const deleteTarget = ref()
const openDeletePopup = async (id) => {
    deletePopupStatus.value = true
    const result = await getDataById(import.meta.env.VITE_URL, id)
    deleteTarget.value = result
}
const deleteConfirm = async () => {
    // Back-end
    const response = await deleteData(import.meta.env.VITE_URL, deleteTarget.value.id)
    // Check Status
    if(response.ok) {
        toastHandle.value = {type: 'success', status: true, message: `The task has been deleted`}
        // Front-end
        taskManagement.value.deleteTask(deleteTarget.value.id)
        deletePopupStatus.value = false
    } else {
        toastHandle.value = {type: 'error', status: true, message: `Task doesn't exist`}
        taskManagement.value.deleteTask(deleteTarget.value.id)
        deletePopupStatus.value = false
    }
}
</script>

<template>
    <DeletePopup v-show="deletePopupStatus" v-if="deleteTarget" :deleteItem="deleteTarget" @close="deletePopupStatus = false" @confirm="deleteConfirm()"/>
    <AddPopup v-show="addPopupStatus" @close="closeAddPopup()" @confirm="confirmAdd"/>
    <EditPopup v-show="editPopupStatus" v-if="targetItem" :itemData="targetItem" @update="updateEdit" @close="closeEditPopup()"/>
    <TaskDetail v-show="taskPopupStatus" v-if="taskDetailTarget" :item="taskDetailTarget" @close="closeTaskDetail"/>
    <div class="w-full min-h-screen p-5">
        <h1 class="flex text-2xl font-bold justify-center mb-5">ITBKK SSA3 Taskboard</h1>
        <div class="flex flex-col space-y-3">
            <div class="flex w-full justify-between font-xl font-bold text-white border-b border-gray-300 p-3 bg-blue-400 rounded">
                <p>Title</p>
                <p>Assignees</p>
                <p>Status</p>
            </div>
            <Toast :toastObject="toastHandle" @close="toastHandle.status = false"/>
            <div class="flex w-full justify-between font-xl ">
                <div @click="openAddPopup()" class="itbkk-button-add w-40 text-center p-2 bg-green-300 rounded cursor-pointer duration-300 hover:bg-green-400 hover:scale-105">
                + Add New Task
                </div>
            </div>
            <div v-for="item in taskManagement.getAllTask()" class="itbkk-item relative flex items-center justify-between w-full p-3 rounded">
                <div class="absolute left-0 w-1 h-10" :class="styleStatus(item.status)"></div>
                <div class="flex items-center space-x-3">
                    <div>
                        <div class="dropdown itbkk-button-action">
                            <p tabindex="0" role="button" class=" btn m-1 border-none font-bold text-2xl">
                                :
                            </p>
                            <ul tabindex="0" class="dropdown-content z-[1] menu p-2 shadow bg-white rounded-box w-40">
                                <li><a class="itbkk-button-edit" @click="openEditPopup(item.id)">Edit</a></li>
                                <li><a class="itbkk-button-delete" @click="openDeletePopup(item.id)">Delete</a></li>
                            </ul>
                        </div>
                    </div>
                    <p class="text-xl font-bold">
                        {{ item.id }}
                    </p>
                    <div class="w-full">
                        <p @click="openTaskDetail(item.id)" class="itbkk-title break-all font-bold text-xl duration-200 cursor-pointer hover:text-gray-700">
                            {{ item.title }}
                        </p>
                        <p class="itbkk-assignees" :class="item.assignees === null ? 'italic text-gray-500' : ''">
                            Assignees : {{ item.assignees === null ? 'Unassigned' : item.assignees }}
                        </p>
                    </div>
                </div>
                <div>
                    <p class="itbkk-status px-4 py-2 rounded" :class="styleStatus(item.status)">{{ item.status }}</p>
                </div>
            </div>
        </div>
    </div>
</template>
