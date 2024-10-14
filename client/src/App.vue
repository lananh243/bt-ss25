<template>
  <div class="task-manager">
    <h2>Quản lý công việc</h2>

    <div class="task-input">
      <input
        type="text"
        placeholder="Nhập tên công việc"
        v-model="newJob.name"
        ref="inputRef"
      />
      <span class="error-message" v-if="error.name">{{ error.name }}</span>
      <button class="add-task-btn" @click="handleAddJob">Thêm công việc</button>
    </div>

    <div class="filter-buttons">
      <button
        :class="{ active: filter === 'all' }"
        @click="setFilter('all')"
        class="filter-btn"
      >
        Tất cả
      </button>
      <button
        :class="{ active: filter === 'completed' }"
        @click="setFilter('completed')"
        class="filter-btn"
      >
        Hoàn thành
      </button>
      <button
        :class="{ active: filter === 'incomplete' }"
        @click="setFilter('incomplete')"
        class="filter-btn"
      >
        Đang thực hiện
      </button>
    </div>
    <!-- Hiệu ứng loading khi đang tải dữ liệu -->
    <div v-if="loading" class="loading">
      <div class="spinner"></div>
      <!-- Biểu tượng loading -->
    </div>
    <div class="task-list-container">
      <ul class="task-list">
        <li class="task-item" v-for="job in filteredJobs" :key="job.id">
          <input
            type="checkbox"
            v-model="job.status"
            @change="handleCheckChange(job)"
          />
          <span
            :style="{ textDecoration: job.status ? 'line-through' : 'none' }"
            >{{ job.name }}</span
          >
          <button class="edit-btn" @click="handleUpdateJob(job)">
            &#9998;
          </button>
          <button class="delete-btn" @click="handleDeleteJob(job.id)">
            &#128465;
          </button>
        </li>
      </ul>
    </div>

    <div class="task-actions">
      <button class="delete-completed-btn" @click="handleDeleteCompletedJobs">
        Xóa công việc hoàn thành
      </button>
      <button class="delete-all-btn" @click="handleDeleteAllJobs">
        Xóa tất cả công việc
      </button>
    </div>
  </div>
</template>

<script setup>
import axios from "axios";
import { computed, onMounted, reactive, ref } from "vue";
const inputRef = ref(null);
const showModal = ref(false);
const jobs = ref([]);
const loading = ref(false);
const filter = ref("all");
const filterStatus = ref("all");

const fetchData = async () => {
  loading.value = true; // Bắt đầu loading
  await new Promise((resolve) => setTimeout(resolve, 2000));
  try {
    const response = await axios.get("http://localhost:8080/jobs");
    jobs.value = response.data;
    loading.value = false;
  } catch (error) {
    loading.value = false;
  }
};

onMounted(() => {
  fetchData();
});
const newJob = reactive({
  name: "",
  status: false,
});

const error = reactive({
  name: "",
});

const clearError = () => {
  error.name = "";
};

const validate = () => {
  let isValid = true;

  if (!newJob.name) {
    error.name = "Tên công việc không được để trống.";
    isValid = false;
  } else if (jobs.value.some((job) => job.name === newJob.name)) {
    error.name = "Tên công việc không được phép trùng.";
    isValid = false;
  }

  return isValid;
};

const handleAddJob = async () => {
  if (!validate()) {
    return;
  }

  try {
    const response = await axios.post("http://localhost:8080/jobs", newJob);
    // Thêm trực tiếp công việc mới vào danh sách hiện tại để tránh gọi lại API
    jobs.value.push(response.data);
    newJob.name = "";
    inputRef.value.focus();
  } catch (error) {}
};

const handleCheckChange = () => {
  // Check if all jobs are completed
  if (jobs.value.every((job) => job.completed)) {
    showModal.value = true; // Show modal if all jobs are done
  }
};
// Lọc công việc theo trạng thái
const setFilter = (newFilter) => {
  filter.value = newFilter;
};

// Lọc sản phẩm
const filteredJobs = computed(() => {
  if (filter.value === "completed") {
    return jobs.value.filter((job) => job.status);
  } else if (filter.value === "incomplete") {
    return jobs.value.filter((job) => !job.status);
  }
  return jobs.value;
});

const handleFilterChange = (status) => {
  filterStatus.value = status; // Cập nhật trạng thái lọc
};

const handleDeleteJob = async (jobId) => {
  const isConfirmed = window.confirm("Bạn có chắc muốn xóa công việc này?");
  if (!isConfirmed) {
    return;
  }

  try {
    await axios.delete(`http://localhost:8080/jobs/${jobId}`);
    jobs.value = jobs.value.filter((job) => job.id !== jobId);
  } catch (error) {
    console.error("Failed to delete job:", error);
  }
};

const handleDeleteCompletedJobs = async () => {
  const isConfirmed = window.confirm(
    "Bạn có chắc muốn xóa tất cả công việc hoàn thành?"
  );
  if (!isConfirmed) {
    return;
  }

  try {
    const completedJobs = jobs.value.filter((job) => job.status);
    for (const job of completedJobs) {
      await axios.delete(`http://localhost:8080/jobs/${job.id}`);
    }
    jobs.value = jobs.value.filter((job) => !job.status);
  } catch (error) {
    console.error("Failed to delete completed jobs:", error);
  }
};

const handleDeleteAllJobs = async () => {
  const isConfirmed = window.confirm("Bạn có chắc muốn xóa tất cả công việc?");
  if (!isConfirmed) {
    return;
  }

  try {
    for (const job of jobs.value) {
      await axios.delete(`http://localhost:8080/jobs/${job.id}`);
    }
    jobs.value = [];
  } catch (error) {
    console.error("Failed to delete all jobs:", error);
  }
};

// Hàm sửa
const handleUpdateJob = async (job) => {
  const updatedJob = { ...job }; // Tạo bản sao công việc để chỉnh sửa

  // Lấy thông tin cập nhật từ người dùng
  updatedJob.title = prompt("Nhập tiêu đề công việc:", job.title);
  try {
    await axios.put(`http://localhost:8080/jobs/${job.id}`, updatedJob);
    // Cập nhật danh sách công việc trong state
    const index = jobs.value.findIndex((j) => j.id === job.id);
    if (index !== -1) {
      jobs.value[index] = updatedJob; // Cập nhật công việc trong danh sách
    }
  } catch (error) {
    console.error("Failed to update job:", error);
  }
};
</script>

<style scoped>
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  font-family: "Arial", sans-serif;
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  background-color: #f7f7f7;
}

.task-manager {
  max-width: 500px;
  width: 100%;
  margin: auto;
  padding: 30px;
  border-radius: 10px;
  margin-top: 80px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  background-color: #fff;
}

h2 {
  text-align: center;
  margin-bottom: 20px;
  font-size: 26px;
  font-weight: bold;
}

.task-input {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-bottom: 20px;
}

.task-input input {
  padding: 12px;
  font-size: 18px;
  border: 1px solid #ccc;
  border-radius: 5px;
}

.error-message {
  color: red;
  font-size: 16px;
}

.add-task-btn {
  padding: 12px;
  background-color: #007bff;
  color: white;
  font-size: 16px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.add-task-btn:hover {
  background-color: #0056b3;
}

.filter-buttons {
  display: flex;
  justify-content: center;
  gap: 10px;
  margin-bottom: 20px;
}

.filter-btn {
  padding: 12px;
  background-color: #f0f0f0;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.filter-btn.active {
  background-color: #007bff;
  color: white;
}

.filter-btn:not(.active):hover {
  background-color: #e0e0e0; /* Màu xám khi hover */
}

.task-list-container {
  max-height: 250px;
  overflow-y: auto;
}

.task-list {
  list-style-type: none;
  margin-bottom: 20px;
}

.task-item {
  display: flex;
  align-items: center;
  gap: 12px;
  justify-content: space-between;
  padding: 12px 0;
  border-bottom: 1px solid #ccc;
}

.task-item input[type="checkbox"] {
  margin-right: 10px;
}

.task-item span {
  flex-grow: 1;
}

.edit-btn,
.delete-btn {
  background-color: transparent;
  border: none;
  cursor: pointer;
  font-size: 20px;
}

.delete-btn {
  color: red;
}

.task-actions {
  display: flex;
  justify-content: space-between;
}

.delete-completed-btn,
.delete-all-btn {
  padding: 12px;
  background-color: red;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.delete-all-btn {
  background-color: #e60000;
}

.delete-all-btn:hover,
.delete-completed-btn:hover {
  background-color: #cc0000;
}

.loading {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100px;
}

.spinner {
  width: 30px;
  height: 30px;
  border: 4px solid #ccc;
  border-top: 4px solid #007bff;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}
</style>
