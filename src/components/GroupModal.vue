<!--
    Editor modal para grupos
-->

<template>
  <BaseModal ref="modalRef" id="groupAddOrEditModal" 
    :title="isAdd ? 'Añadiendo nuevo grupo' : 'Editando grupo'"
    :name="name">
    <template #body>
      <form id="addOrEditGroupForm" @submit.prevent="e => setGroup()">
        <div class="container">
          <TextBox :start="group.name" id="e-name" label="Name"
            @change="(v) => name = gState.resolve(group.subjectId).short + ':' + v" />
          <SelectBox :all="gState.model.getSubjects()" 
            :start="group.subjectId" :displayCol="'short'" id="e-subjectId"
            label="Asignatura" />
          <br>
          <TextBox :start="'' + group.credits" id="e-credits" label="Créditos" />
          <TextBox :start="'' + group.isLab" id="e-isLab" label="Prácticas" />
          <SelectBox 
            :all="gState.model.getUsers({ userRole: gState.model.UserRole.TEACHER })" 
            :start="group.teacherId"
            :displayCol="'userName'" id="e-teacherId" label="Profesor" />
          <br>
          <SlotBox 
            :start="group.slots"
            id="e-slots" label="Slots" />
          <button 
              type="button" 
              class="btn btn-secondary mt-2" 
              @click="assignLabSlot"
            >
              Asignar Laboratorio (2h)
            </button>
            <button 
              type="button" 
              class="btn btn-secondary mt-2" 
              @click="assignClassSlot"
            >
              Asignar Aula Teórica (1h)
          </button>
        </div>
        <button type="submit" class="invisible">Submit</button>
      </form>
    </template>
    <template #footer>
      <button @click.prevent="() => setGroup()" class="btn btn-primary">
        {{ isAdd ? 'Añadir grupo' : 'Confirmar cambios a' }}
        <span class="name">{{ name }}</span>
      </button>
    </template>
  </BaseModal>
</template>

<script setup>

import BaseModal from './BaseModal.vue';
import TextBox from './TextBox.vue'
import SelectBox from './SelectBox.vue'
import SlotBox from './SlotBox.vue'

import { gState } from '../state.js';
import { ref } from 'vue'

const emit = defineEmits(['add', 'edit'])

const props = defineProps({
  group: {
    type: Object,
    default: () => ({
      id: -1,
      name: "",
      subjectId: null,
      credits: 0,
      isLab: false,
      teacherId: null,
      slots: [], // Inicializa slots como un array vacío
    }),
  },
  isAdd: Boolean,
});


let modalRef = ref(null);
let name = ref(gState.resolve(props.group.subjectId).short + ':' + props.group.name);

function setGroup() {
  const group = props.group;
  const form = document.getElementById("addOrEditGroupForm");

  const inputFor = (name) => {
    const input = form.querySelector(`input[name=${name}]`);
    if (!input) console.error("ERROR: no input for", name);
    return input;
  };

  const valueFor = (name) => inputFor(name)?.value || "";

  console.log("Guardando grupo...", group);

  // Asigna un ID al grupo si no tiene uno
  if (!group.id || group.id < 0) {
    group.id = gState.model.getNextGroupId();
    console.log("ID asignado al grupo:", group.id);
  }

  // Procesa los slots
  const slots = JSON.parse(valueFor("e-slots") || "[]");
  const otherSlots = gState.model.getSlots();
  const subject = gState.model.resolve(+valueFor("e-subjectId"));

  for (let slot of slots) {
    slot.semester = subject.semester;
    slot.groupId = group.id;

    if (!slot.id) {
      // Crea un nuevo slot
      slot.id = gState.model.addSlot(slot);
    } else {
      // Actualiza un slot existente
      gState.model.setSlot(slot);
    }
  }

  // Verifica si los slots causan conflictos
  for (let slot of slots) {
    inputFor(`e-slots-${slot.id}-location`)?.setCustomValidity(
      gState.model.overlapsOtherSlots(slot, otherSlots, true)
        ? "Conflicto con otros horarios"
        : ""
    );
  }

  // Verifica la validez del formulario antes de guardar
  if (!form.checkValidity()) {
    console.warn("El formulario contiene errores. Mostrando mensajes de error.");
    form.querySelector("button[type=submit]")?.click();
    return;
  }

  // Crea el grupo actualizado
  const updatedGroup = new gState.model.Group(
    group.id,
    valueFor("e-name"),
    valueFor("e-subjectId") ? +valueFor("e-subjectId") : undefined,
    +valueFor("e-credits"),
    valueFor("e-isLab") === "true",
    slots.map((s) => s.id),
    valueFor("e-teacherId") ? +valueFor("e-teacherId") : undefined
  );

  // Lanza el evento correspondiente
  emit(props.isAdd ? "add" : "edit", updatedGroup);

  // Cierra el modal
  modalRef.value.hide();
}


// para que el padre pueda llamar a show (hide no debería hacer falta)
function show() {
  modalRef.value.show();
}
defineExpose({ show });

function assignLabSlot() {
  const group = props.group;

  // Encuentra una franja de 2 horas disponible
  const availableSlot = findAvailableSlot(2); // Duración de 2h
  if (availableSlot) {
    // Crea un nuevo array para asegurar reactividad
    group.slots = [
      ...group.slots,
      {
        weekDay: availableSlot.weekDay,
        startTime: availableSlot.startTime,
        endTime: availableSlot.endTime,
        location: "Laboratorio",
      },
    ];
    console.log("Franja de laboratorio asignada:", availableSlot);
  } else {
    console.error("No hay franjas disponibles para laboratorio.");
  }
}


function assignClassSlot() {
  const group = props.group;

  // Encuentra una franja de 1 hora disponible
  const availableSlot = findAvailableSlot(1); // Duración de 1h
  if (availableSlot) {
    group.slots = [
      ...group.slots,
      {
        weekDay: availableSlot.weekDay,
        startTime: availableSlot.startTime,
        endTime: availableSlot.endTime,
        location: "Laboratorio",
      },
    ];
    console.log("Franja de aula teórica asignada:", availableSlot);
  } else {
    console.error("No hay franjas disponibles para aula teórica.");
  }
}

function findAvailableSlot(duration) {
  const startHour = 8; // Hora de inicio
  const endHour = 18; // Hora de fin
  const step = 1; // Incremento de horas
  const days = ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"];

  for (const day of days) {
    for (let hour = startHour; hour <= endHour - duration; hour += step) {
      const startTime = hour * 100; // Convierte horas a formato 0800, 0900, etc.
      const endTime = (hour + duration) * 100;

      // Verifica conflictos con slots existentes
      const hasConflict = gState.model.getSlots().some(slot => 
        slot.weekDay === day &&
        ((startTime >= slot.startTime && startTime < slot.endTime) ||
         (endTime > slot.startTime && endTime <= slot.endTime))
      );

      if (!hasConflict) {
        return { weekDay: day, startTime, endTime };
      }
    }
  }

  // No se encontró una franja disponible
  return null;
}

</script>

<style scoped>
.name {
  font-weight: 200%;
}
</style>