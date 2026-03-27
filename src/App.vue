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

const carreras = [
  'Ingeniería en Sistemas Computacionales',
  'Ingeniería Mecatrónica',
  'Ingeniería Civil',
  'Ingeniería en Gestión Empresarial',
  'Ingeniería Industrial',
  'Licenciatura en Administración',
  'Licenciatura en Arquitectura'
];

const selectedCarrera = ref(carreras[0]);
const searchQuery = ref('');
const currentPage = ref(1);
const pageSize = 15;

const countAlumnosByCarrera = (carrera) => {
  return alumnos.value.filter(a => a.carrera === carrera).length;
};

import { computed } from 'vue';

const alumnosFiltrados = computed(() => {
  if (searchQuery.value.trim()) {
    const query = searchQuery.value.toLowerCase();
    return alumnos.value.filter(a => 
      (a.nombre || '').toLowerCase().includes(query) ||
      (a.apellidoPaterno || '').toLowerCase().includes(query) ||
      (a.apellidoMaterno || '').toLowerCase().includes(query) ||
      (a.numeroControl || '').toLowerCase().includes(query)
    );
  }
  return alumnos.value.filter(a => a.carrera === selectedCarrera.value);
});

const totalPages = computed(() => {
  return Math.ceil(alumnosFiltrados.value.length / pageSize);
});

const alumnosPaginados = computed(() => {
  const start = (currentPage.value - 1) * pageSize;
  const end = start + pageSize;
  return alumnosFiltrados.value.slice(start, end);
});

const cambiarCarrera = (carrera) => {
  selectedCarrera.value = carrera;
  currentPage.value = 1;
};

const cargarAlumnos = async () => {

  const response = await axios.get('backalumnos-production.up.railway.app/alumnos/traer-alumnos')
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
      await axios.put(`backalumnos-production.up.railway.app/alumnos/editar-alumnos/${nuevoAlumno.value.id}`, payload);
      Swal.fire({
        icon: 'success',
        title: 'Alumno actualizado correctamente',
        showConfirmButton: false,
        timer: 1500
      });
      editado.value = false;
    } else {
      await axios.post('backalumnos-production.up.railway.app/alumnos/insertar-alumnos', payload);
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
    await axios.delete(`backalumnos-production.up.railway.app/alumnos/eliminar-alumnos/${id}`);

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
                  <option v-for="carrera in carreras" :key="carrera" :value="carrera">{{ carrera }}</option>
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

      <div class="col-md-12 mb-4">
        <div class="row justify-content-center mb-3">
          <div class="col-auto">
            <div class="badge bg-primary rounded-pill px-4 py-2 shadow-sm d-flex align-items-center gap-2" style="font-size: 1.1rem; background-color: #2563eb !important;">
              <i class="bi bi-people-fill"></i>
              <span>Total de Estudiantes: {{ alumnos.length }}</span>
            </div>
          </div>
        </div>
        <div class="row justify-content-center mb-4">
          <div class="col-md-6">
            <div class="input-group shadow-sm" style="border-radius: 20px; overflow: hidden; border: 1px solid #e2e8f0;">
              <span class="input-group-text bg-white border-0 ps-3">
                <i class="bi bi-search text-primary"></i>
              </span>
              <input 
                type="text" 
                class="form-control border-0 py-2" 
                v-model="searchQuery" 
                placeholder="Buscar alumno por nombre, apellidos o número de control..."
                style="box-shadow: none;"
              >
            </div>
          </div>
        </div>

        <div class="d-flex flex-wrap justify-content-center gap-2 mb-4">
          <button v-for="carrera in carreras" :key="carrera" 
            @click="cambiarCarrera(carrera)"
            :class="['btn', selectedCarrera === carrera ? 'btn-primary' : 'btn-outline-primary']"
            style="border-radius: 12px; font-weight: 500; transition: all 0.3s ease; display: flex; align-items: center; gap: 8px;">
            {{ carrera }}
            <span class="badge rounded-pill bg-primary" v-if="selectedCarrera === carrera">{{ countAlumnosByCarrera(carrera) }}</span>
            <span class="badge rounded-pill bg-primary" style="opacity: 0.8;" v-else>{{ countAlumnosByCarrera(carrera) }}</span>
          </button>
        </div>

        <div class="card shadow" style="border-radius: 15px; border: none;">
          <div class="card-body">
            <div class="d-flex justify-content-between align-items-center mb-3">
              <h5 class="card-title m-0" style="color: #2c3e50; font-weight: bold;">
                <i class="bi bi-mortarboard-fill me-2"></i> 
                {{ searchQuery ? 'Resultados de búsqueda' : selectedCarrera }}
              </h5>
              <span class="badge bg-primary rounded-pill px-3 py-2">{{ alumnosFiltrados.length }} alumnos</span>
            </div>
            <div class="table-responsive">
              <table class="table table-hover align-middle">
                <thead class="table-light">
                  <tr>
                    <th scope="col">#</th>
                    <th scope="col">NOMBRE</th>
                    <th scope="col">AP. PATERNO</th>
                    <th scope="col">AP. MATERNO</th>
                    <th scope="col">EMAIL</th>
                    <th scope="col">Nº CONTROL</th>
                    <th scope="col">CARRERA</th>
                    <th scope="col">TELÉFONO</th>
                    <th scope="col">IMAGEN</th>
                    <th scope="col">ACCIONES</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="(alumno, index) in alumnosPaginados" :key="alumno.id">
                    <td>{{ (currentPage - 1) * pageSize + index + 1 }}</td>
                    <td>{{ alumno.nombre }}</td>
                    <td>{{ alumno.apellidoPaterno }}</td>
                    <td>{{ alumno.apellidoMaterno }}</td>
                    <td>{{ alumno.email }}</td>
                    <td>{{ alumno.numeroControl }}</td>
                    <td><small class="text-muted">{{ alumno.carrera }}</small></td>
                    <td>{{ alumno.telefono }}</td>
                    <td><img :src="alumno.imagenURL || defaultImage" alt="Alumno" width="40" height="40" class="rounded-circle shadow-sm" style="object-fit: cover;"></td>
                    <td>
                      <div class="d-flex gap-2">
                        <button @click="editarAlumnos(alumno)" class="btn btn-warning btn-sm text-white" style="background-color: #f59e0b; border: none;">
                          <i class="bi bi-pencil-fill"></i>
                        </button>
                        <button @click="eliminarAlumno(alumno.id)" class="btn btn-danger btn-sm" style="background-color: #ef4444; border: none;">
                          <i class="bi bi-trash3-fill"></i>
                        </button>
                      </div>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>

            <!-- Pagination -->
            <nav v-if="totalPages > 1" class="mt-4">
              <ul class="pagination justify-content-center">
                <li class="page-item" :class="{ disabled: currentPage === 1 }">
                  <button class="page-link" @click="currentPage--" aria-label="Previous">
                    <span aria-hidden="true">&laquo; Anterior</span>
                  </button>
                </li>
                <li v-for="page in totalPages" :key="page" class="page-item" :class="{ active: currentPage === page }">
                  <button class="page-link" @click="currentPage = page">{{ page }}</button>
                </li>
                <li class="page-item" :class="{ disabled: currentPage === totalPages }">
                  <button class="page-link" @click="currentPage++" aria-label="Next">
                    <span aria-hidden="true">Siguiente &raquo;</span>
                  </button>
                </li>
              </ul>
            </nav>
          </div>
        </div>
      </div>
    </div>

  </div>

</template>

<style scoped></style>
