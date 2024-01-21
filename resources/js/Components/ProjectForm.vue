<template>
  <div class="project-form-container">
    <h2 class="form-title">{{ op === 1 ? 'Crear ' : 'Editar ' }} Proyecto</h2>
    <form @submit.prevent="(op === 1 ? save() : update())" class="project-form">
      <div class="form-group">
        <label for="name" class="form-label">Nombre:</label>
        <input
          type="text"
          id="name"
          v-model="form.name"
          class="form-input"
          required
        />
      </div>

      <div class="form-group">
        <label for="description" class="form-label">Descripción:</label>
        <input
          type="text"
          id="description"
          v-model="form.description"
          class="form-input"
        />
      </div>

      <div class="form-group">
        <label for="date_end" class="form-label">Fecha de finalización:</label>
        <input
          type="date"
          id="date_end"
          v-model="form.date_end"
          class="form-input"
        />
      </div>

      <div class="form-actions">
        <button
          type="button"
          @click="cerrar"
          class="form-btn form-btn-cancel"
        >
          Cancelar
        </button>
        <button
          type="submit"
          :disabled="form.processing"
          class="form-btn form-btn-ok"
        >
          OK
        </button>
      </div>
    </form>
  </div>
</template>

<script setup>
import { useForm } from '@inertiajs/vue3';

const props = defineProps({
  closeForm: { type: Function },
  project: { type: Object, default: () => ({}) },
  user_id: { type: Number },
  op: { type: Number }
});

const form = useForm({
  name: props.project.name,
  description: props.project.description,
  date_end: props.project.date_end,
  user_id: props.user_id
});

const cerrar = () => {
  form.reset();
  props.closeForm();
};

const save = () => {
  form.user_id = props.user_id;
  form.post(route('projects.store'), {
    onSuccess: () => cerrar()
  });
};

const update = () => {
  form.put(route('projects.update', props.project.id), {
    onSuccess: () => cerrar()
  });
};
</script>

<style scoped>
.project-form-container {
  max-width: 400px;
  margin: 0 auto;
  background-color: #fff;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
}

.form-title {
  text-align: center;
  font-weight: bold;
  margin-bottom: 20px;
}

.project-form {
  display: flex;
  flex-direction: column;
}

.form-group {
  margin-bottom: 15px;
}

.form-label {
  font-size: 14px;
  margin-bottom: 5px;
  color: #333;
}

.form-input {
  width: 100%;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 4px;
  outline: none;
}

.form-actions {
  display: flex;
  justify-content: space-between;
  margin-top: 20px;
}

.form-btn {
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  outline: none;
}

.form-btn-cancel {
  background-color: #e74c3c;
  color: #fff;
}

.form-btn-ok {
  background-color: green;
  color: #fff;
}
</style>
