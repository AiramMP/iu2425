<template>
    <h4>Usuario <span class="name">{{ user.firstName }} {{ user.lastName }}</span></h4>

    <div class="form-check">
        <input 
          class="form-check-input" 
          type="checkbox" 
          id="toggleSemester" 
          v-model="showSecondSemester" 
          @change="filterBySemester"
        />
        <label class="form-check-label" for="toggleSemester">
          Mostrar Segundo Cuatrimestre
        </label>
    </div>

    <table>
        <tbody>
            <tr>
                <th>Rol</th>
                <td>{{ user.userRole }} </td>
            </tr>
            <tr>
                <th>Créditos</th>
                <td>imparte {{ user.assignedCredits }} de {{ user.maxCredits }}</td>
            </tr>
            <tr>
                <th>Grupos a los que da clase</th>
                <td v-if="user.groups.length">
                    <span v-for="group in user.groups.map(g => formatNiceGroup(g))" v-bind:key="group">
                    <span class="badge bg-primary mx-1">
                        {{ group }}
                    </span>
                    </span>
                </td>
                <td v-else> (ninguno) </td>
            </tr>
        </tbody>
    </table>

    <SortableGrid :data="addSlotCols(filteredSlots)" :columns="slotColumns" 
      :filter="{ all: '', fields: [] }" v-model:sorter="sorter" />
    <TimeTable :slots="filteredSlots" />

    <h5>Acciones</h5>
    <div class="btn-group">
        <button @click="$emit('editUser')" class="btn btn-outline-success" title="editar Usuario">✏️ Editar</button>
        <button @click="$emit('rmUser')" class="btn btn-outline-danger" title="Eliminar Usuario">🗑️ Eliminar</button>
    </div>
</template>


<script setup>
import SortableGrid from './SortableGrid.vue';
import TimeTable from './TimeTable.vue';

import { gState, semesterNames, weekDayNames } from '../state.js';
import { ref, computed } from 'vue';

defineEmits([
  'filterUser',
  'editUser',
  'rmUser',
]);

const props = defineProps({
    user: Object // see definition of User in ../model.js
});

let sorter = ref([{key: "weekDay", order: 1}]);

// Nueva variable reactiva para controlar el cuatrimestre mostrado
let showSecondSemester = ref(false); // Por defecto, mostrar el primer cuatrimestre

// Filtra los slots según el cuatrimestre seleccionado
const filteredSlots = computed(() => {
    const result = slots.value.filter(slot =>
        showSecondSemester.value 
            ? slot.semester === "SPRING" 
            : slot.semester === "FALL"
    );
    console.log("Slots filtrados:", result);
    return result;
});


// Lógica para registrar el cambio de cuatrimestre
function filterBySemester() {
    console.log(
        "Mostrando slots del cuatrimestre:", 
        showSecondSemester.value ? "Segundo" : "Primero"
    );
}

// Lógica existente para los slots
const slots = computed(() => slotsOfAllGroups(props.user.groups));

const slotsOfAllGroups = (gg) => {
    const rv = [];
    for (let g of gg.map(o => gState.resolve(o))) {
        for (let s of g.slots.map(s => gState.resolve(s))) {
            rv.push(s);
        }
    }
    console.log("Slots recuperados:", rv);
    return rv;
};


const slotColumns = [
    { key: 'niceGroup', display: 'Grupo', type: 'String' },
    {
        key: 'weekDay', display: 'Día', type: 'Enum',
        values: gState.model.WeekDay,
        displayValues: weekDayNames
    },
    {
        key: 'semester', display: 'Cuatr.', type: 'Enum',
        values: gState.model.Semester,
        displayValues: semesterNames
    },
    { key: 'niceStart', display: 'Inicio', type: 'Time' },
    { key: 'niceEnd', display: 'Final', type: 'Time' },
    { key: 'location', display: 'Lugar', type: 'String' },
];

const addSlotCols = (ss) => {
    console.log("Procesando slots en addSlotCols:", ss);
    for (let s of ss) {
        s.niceGroup = formatNiceGroup(s.groupId);
        s.niceStart = formatTime(s.startTime);
        s.niceEnd = formatTime(s.endTime);
    }
    console.log("Slots procesados:", ss);
    return ss;
};


// 1450 => "14:50"
const formatTime = t => `${Math.floor(t / 100)}:` + `0${t % 100}`.slice(-2);

const formatNiceGroup = groupId => {
    const group = gState.resolve(groupId);
    const subject = gState.resolve(group.subjectId);
    return `${subject.short}:${group.name}`;
};
</script>
