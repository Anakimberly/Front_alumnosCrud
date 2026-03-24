<script setup>
import { ref, onMounted } from 'vue';
import axios from 'axios';
import Swal from 'sweetalert2';

const alumnos = ref([]);

const nuevoAlumno = ref({
  nombre: '',
  apellidoPaterno: '',
  apellidoMaterno: '',
  email: '',
  numeroControl: '',
  carrera: '',
  telefono: '',
  telefonoPrefijo: 'MX +52',
  imagenURL: ''
});

const editado = ref(false);

const cargarAlumnos = async () => {

  const response = await axios.get('http://localhost:8080/alumnos/traer-alumnos')
  alumnos.value = response.data;
  console.log(alumnos.value);
}
const defaultImage = 'https://cdn-icons-png.flaticon.com/512/3135/3135715.png';

const agregarAlumno = async () => {
  // Validaciones
  const nombreRegex = /^[a-zA-ZáéíóúÁÉÍÓÚñÑ\s]+$/;
  if (!nombreRegex.test(nuevoAlumno.value.nombre)) {
    Swal.fire('Error', 'El nombre solo debe contener letras', 'error');
    return;
  }
  if (!nombreRegex.test(nuevoAlumno.value.apellidoPaterno) || !nombreRegex.test(nuevoAlumno.value.apellidoMaterno)) {
    Swal.fire('Error', 'Los apellidos solo deben contener letras', 'error');
    return;
  }
  if (!nuevoAlumno.value.email.includes('@')) {
    Swal.fire('Error', 'El email debe contener un @', 'error');
    return;
  }
  if (!/^26\d{6}$/.test(nuevoAlumno.value.numeroControl)) {
    Swal.fire('Error', 'El número de control debe tener 8 dígitos e iniciar con 26', 'error');
    return;
  }
  if (!/^\d{10}$/.test(nuevoAlumno.value.telefono)) {
    Swal.fire('Error', 'El teléfono debe tener exactamente 10 dígitos', 'error');
    return;
  }

  // Imagen por defecto
  if (!nuevoAlumno.value.imagenURL) {
    nuevoAlumno.value.imagenURL = defaultImage;
  }

  const payload = {
    ...nuevoAlumno.value,
    telefono: `${nuevoAlumno.value.telefonoPrefijo} ${nuevoAlumno.value.telefono}`
  };

  try {
    if (editado.value) {
      await axios.put(`http://localhost:8080/alumnos/editar-alumnos/${nuevoAlumno.value.id}`, payload);
      Swal.fire({
        icon: 'success',
        title: 'Alumno actualizado correctamente',
        showConfirmButton: false,
        timer: 1500
      });
      editado.value = false;
    } else {
      await axios.post('http://localhost:8080/alumnos/insertar-alumnos', payload);
      Swal.fire({
        icon: 'success',
        title: 'Alumno agregado correctamente',
        showConfirmButton: false,
        timer: 1500
      });
    }

    await cargarAlumnos();
    limpiarFormulario();
  } catch (error) {
    if (error.response && error.response.status === 400) {
      Swal.fire({
        icon: 'error',
        title: 'Error de Validación',
        text: error.response.data
      });
    } else {
      Swal.fire({
        icon: 'error',
        title: 'Hubo un error',
        text: 'Ocurrió un error inesperado al procesar la solicitud.'
      });
    }
  }
}

const limpiarFormulario = () => {
  nuevoAlumno.value = {
    nombre: '',
    apellidoPaterno: '',
    apellidoMaterno: '',
    email: '',
    numeroControl: '',
    carrera: '',
    telefono: '',
    telefonoPrefijo: 'MX +52',
    imagenURL: ''
  };
  editado.value = false;
}
// Función para editar un alumno
const editarAlumnos = (alumno) => {
  const [prefijo, ...telefonoRest] = (alumno.telefono || '').split(' ');
  // Handle case where there's no space
  if (telefonoRest.length > 0) {
    nuevoAlumno.value = {
      ...alumno,
      telefonoPrefijo: prefijo + ' ' + telefonoRest[0],
      telefono: telefonoRest.slice(1).join(' ')
    };
  } else {
    Object.assign(nuevoAlumno.value, alumno);
    nuevoAlumno.value.telefonoPrefijo = 'MX +52';
  }
  editado.value = true
}

const eliminarAlumno = async (id) => {

  Swal.fire({
    title: '¿Estás seguro de eliminar el alumno?',
    text: "No podrás revertir esto!",
    icon: 'warning',
    showCancelButton: true,
    confirmButtonColor: '#3085d6',
    cancelButtonColor: '#d33',
    confirmButtonText: 'Sí, eliminarlo!'
  }).then(async (result) => {
    if (result.isConfirmed) {
      await eliminarAlumnoPorId(id);
      Swal.fire(
        'Eliminado!',
        'El alumno ha sido eliminado.',
        'success'
      )
    }
  })
}

const eliminarAlumnoPorId = async (id) => {
  try {
    await axios.delete(`http://localhost:8080/alumnos/eliminar-alumnos/${id}`);

    console.log('Alumno eliminado con id:', id);
    await cargarAlumnos();
  } catch (errr) {
    console.error('Error al eliminar el alumno:', errr);
    Swal.fire({
      icon: 'error',
      title: 'Error al eliminar el alumno',
      text: 'No se pudo eliminar el alumno.',

    });
  }
}
onMounted(cargarAlumnos);



</script>

<template>
  <div class="container">
    <div class="row">
      <div class="col-md-12 mt-4">
        <div class="card shadow p-4 mb-4" style="border-radius: 15px; border: none;">
          <h2 class="text-center mb-4" style="color: #2c3e50; font-weight: bold; color: #1e3a8a;">Formulario de Alumnos</h2>
          <form @submit.prevent="agregarAlumno">
            <div class="row">
              <div class="col-md-6 mb-3">
                <label for="nombre" class="form-label" style="font-weight: 500;">Nombre</label>
                <input type="text" class="form-control" id="nombre" v-model="nuevoAlumno.nombre" required>
              </div>
              <div class="col-md-6 mb-3">
                <label for="apellidoPaterno" class="form-label" style="font-weight: 500;">Apellido Paterno</label>
                <input type="text" class="form-control" id="apellidoPaterno" v-model="nuevoAlumno.apellidoPaterno" required>
              </div>
              <div class="col-md-6 mb-3">
                <label for="apellidoMaterno" class="form-label" style="font-weight: 500;">Apellido Materno</label>
                <input type="text" class="form-control" id="apellidoMaterno" v-model="nuevoAlumno.apellidoMaterno" required>
              </div>
              <div class="col-md-6 mb-3">
                <label for="email" class="form-label" style="font-weight: 500;">Email</label>
                <input type="email" class="form-control" id="email" v-model="nuevoAlumno.email" required>
              </div>
              <div class="col-md-6 mb-3">
                <label for="numeroControl" class="form-label" style="font-weight: 500;">Número de Control</label>
                <input type="text" class="form-control" id="numeroControl" v-model="nuevoAlumno.numeroControl" required maxlength="8">
                <small class="text-danger" v-if="nuevoAlumno.numeroControl && !/^26\d{6}$/.test(nuevoAlumno.numeroControl)">
                  Debe tener exactamente 8 dígitos comenzando con 26
                </small>
              </div>
              <div class="col-md-6 mb-3">
                <label for="carrera" class="form-label" style="font-weight: 500;">Carrera</label>
                <select class="form-select" id="carrera" v-model="nuevoAlumno.carrera" required>
                  <option value="" disabled selected>-- Selecciona una carrera --</option>
                  <option value="Ingeniería en Sistemas">Ingeniería en Sistemas</option>
                  <option value="Ingeniería Mecatrónica">Ingeniería Mecatrónica</option>
                  <option value="Ingeniería Civil">Ingeniería Civil</option>
                  <option value="Ingeniería en Gestión Empresarial">Ingeniería en Gestión Empresarial</option>
                  <option value="Ingeniería Industrial">Ingeniería Industrial</option>
                  <option value="Licenciatura en Administración">Licenciatura en Administración</option>
                  <option value="Licenciatura en Arquitectura">Licenciatura en Arquitectura</option>
                </select>
              </div>
              <div class="col-md-6 mb-3">
                <label for="telefono" class="form-label" style="font-weight: 500;">Teléfono</label>
                <div class="input-group">
                  <select class="form-select" style="max-width: 120px;" v-model="nuevoAlumno.telefonoPrefijo">
                    <option value="MX +52">🇲🇽 +52</option>
                    <option value="US +1">🇺🇸 +1</option>
                    <option value="CA +1">🇨🇦 +1</option>
                    <option value="ES +34">🇪🇸 +34</option>
                    <option value="AR +54">🇦🇷 +54</option>
                    <option value="CO +57">🇨🇴 +57</option>
                    <option value="CL +56">🇨🇱 +56</option>
                  </select>
                  <input type="text" class="form-control" id="telefono" v-model="nuevoAlumno.telefono" required placeholder="10 dígitos" maxlength="10">
                </div>
              </div>
              <div class="col-md-12 mb-3">
                <label for="imagenURL" class="form-label" style="font-weight: 500;">Imagen URL</label>
                <input type="text" class="form-control" id="imagenURL" v-model="nuevoAlumno.imagenURL">
              </div>
            </div>

            <div class="d-flex gap-2">
              <button type="submit" class="btn btn-primary px-4" style="background-color: #2563eb; border: none; border-radius: 8px;">
                {{ editado ? 'Actualizar Alumno' : 'Agregar Alumno' }}
              </button>
              <button type="button" class="btn btn-secondary px-4" @click="limpiarFormulario" style="background-color: #4b5563; border: none; border-radius: 8px;">
                Limpiar
              </button>
            </div>
          </form>

        </div>

      </div>

      <div class="col-md-12">
        <div class="card shadow" style="border-radius: 15px; border: none;">
          <div class="card-body">
            <h5 class="card-title mb-3" style="color: #2c3e50; font-weight: bold;">Lista de Alumnos</h5>
            <table class="table table-hover align-middle">
              <thead class="table-light">
                <tr>
                  <th scope="col">Num. Control</th>
                  <th scope="col">Nombre</th>
                  <th scope="col">Apellidos</th>
                  <th scope="col">Email</th>
                  <th scope="col">Carrera</th>
                  <th scope="col">Teléfono</th>
                  <th scope="col">Imagen</th>
                  <th scope="col">Acciones</th>

                </tr>
              </thead>
              <tbody>
                <tr v-for="alumno in alumnos" :key="alumno.id">
                  <td>{{ alumno.numeroControl }}</td>
                  <td>{{ alumno.nombre }}</td>
                  <td>{{ alumno.apellidoPaterno }} {{ alumno.apellidoMaterno }}</td>
                  <td>{{ alumno.email }}</td>
                  <td>{{ alumno.carrera }}</td>
                  <td>{{ alumno.telefono }}</td>
                  <td><img :src="alumno.imagenURL || defaultImage" alt="Alumno" width="40" class="rounded-circle shadow-sm"></td>
                  <td>
                    <button @click="editarAlumnos(alumno)" class="btn btn-warning btn-sm me-2"><i class="bi bi-pencil-fill"></i></button>
                    <button @click="eliminarAlumno(alumno.id)" class="btn btn-danger btn-sm"><i class="bi bi-trash3-fill"></i></button>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </div>

  </div>

</template>

<style scoped></style>
