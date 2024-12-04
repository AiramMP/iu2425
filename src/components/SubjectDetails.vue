<template>
<h4>Asignatura <span class="name">{{ subject.short }}</span></h4>
    <table>
      <tbody>
        <tr>
          <th>Nombre</th>
          <td>{{ subject.name }} </td>
        </tr>
        <tr>
          <th>Créditos</th>
          <td>{{ subject.credits }} </td>
        </tr>
        <tr>
          <th>Cuatrimestre</th>
          <td>{{ subject.semester }} </td>
        </tr>
        <tr>
          <th>Códigos GEA</th>
          <td>{{ subject.codes.replaceAll('-', ', ') }} </td>
        </tr>
        <tr>
          <th>Grupos</th>
          <td v-if="subject.groups.length">
             <span v-for="group in subject.groups.map(g => gState.resolve(g).name)" v-bind:key="group">
              <span class="badge bg-primary mx-1">
                {{ group }}
              </span>
             </span>
              
          </td>
          <td v-else> (ninguno) </td>
        </tr>
      </tbody>
    </table>

    <h5>Acciones</h5>
    <div class="btn-group">
      <button @click="$emit('editSubject')" class="btn btn-outline-success" title="editar Asignatura">✏️ Editar</button>
      <button @click="$emit('rmSubject')" class="btn btn-outline-danger" title="Eliminar Asignatura">🗑️ Eliminar</button>
    </div>
</template>

<script setup>

import { gState } from '../state.js';

defineEmits([
  'editSubject',
  'rmSubject',
])

defineProps({
  subject: Object // see definition of Subject in ../model.js
})

</script>