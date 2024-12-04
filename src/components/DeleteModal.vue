<template>
    <BaseModal ref="modalRef" id="userAddOrEditModal"
      :title="'Borrando '" 
      :name="name">
      <template #body>
        <form id="addOrEditUserForm"
          @submit.prevent="e => deleteElem()">
          <div class="container">
            <p class="text-danger fs-5">¿Estás seguro de que quieres borrar {{ name }}? ¡Esta acción es <b>irreversible</b>!</p>
          </div>
          <button type="submit" class="invisible">Submit</button>
        </form>
      </template>
      <template #footer>
        <button @click.prevent="() => deleteElem()" class="btn btn-danger">
          Borrar {{ name }}
        </button>
      </template>
    </BaseModal>
</template>

<script setup>

import BaseModal from './BaseModal.vue';

import { gState } from '../state.js';
import { ref } from 'vue'

const emit = defineEmits(['delete', 'cancel'])

const props = defineProps({
  elem : Object
})

let modalRef = ref(null);
let name = ref((typeof props.elem.subjectId !== 'undefined' ? (gState.resolve(props.elem.subjectId).short + ':') : '') + (typeof props.elem.userName === 'undefined' ? props.elem.name : (props.elem.firstName + ' ' + props.elem.lastName)));

function deleteElem() {    

  // todo válido: lanza evento a padre, y cierra modal
  emit('delete', props.elem.id);
  modalRef.value.hide() 
}

// para que el padre pueda llamar a show (hide no debería hacer falta)
function show() {
  modalRef.value.show();
}
defineExpose({ show });
</script>
