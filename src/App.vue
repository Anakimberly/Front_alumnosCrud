<script setup>
import { ref, onMounted } from 'vue';
import axios from 'axios';
import Swal from 'sweetalert2';

const API = "http://3.139.61.49:5000";

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
  const response = await axios.get(`${API}/alumnos/traer-alumnos`);
  alumnos.value = response.data;
  console.log(alumnos.value);
}

const defaultImage = 'https://cdn-icons-png.flaticon.com/512/3135/3135715.png';

const agregarAlumno = async () => {
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

  if (!nuevoAlumno.value.imagenURL) {
    nuevoAlumno.value.imagenURL = defaultImage;
  }

  const payload = {
    ...nuevoAlumno.value,
    telefono: `${nuevoAlumno.value.telefonoPrefijo} ${nuevoAlumno.value.telefono}`
  };

  try {
    if (editado.value) {
      await axios.put(`${API}/alumnos/editar-alumnos/${nuevoAlumno.value.id}`, payload);
      Swal.fire({ icon: 'success', title: 'Alumno actualizado correctamente', showConfirmButton: false, timer: 1500 });
      editado.value = false;
    } else {
      await axios.post(`${API}/alumnos/insertar-alumnos`, payload);
      Swal.fire({ icon: 'success', title: 'Alumno agregado correctamente', showConfirmButton: false, timer: 1500 });
    }

    await cargarAlumnos();
    limpiarFormulario();
  } catch (error) {
    if (error.response && error.response.status === 400) {
      Swal.fire({ icon: 'error', title: 'Error de Validación', text: error.response.data });
    } else {
      Swal.fire({ icon: 'error', title: 'Hubo un error', text: 'Ocurrió un error inesperado al procesar la solicitud.' });
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

const editarAlumnos = (alumno) => {
  const [prefijo, ...telefonoRest] = (alumno.telefono || '').split(' ');
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
  editado.value = true;
}

const eliminarAlumno = async (id) => {
  Swal.fire({
    title: '¿Estás seguro de eliminar el alumno?',
    text: "No podrás revertir esto!",
    icon: 'warning',
    showCancelButton: true,
    confirmButtonText: 'Sí, eliminarlo!'
  }).then(async (result) => {
    if (result.isConfirmed) {
      await eliminarAlumnoPorId(id);
      Swal.fire('Eliminado!', 'El alumno ha sido eliminado.', 'success');
    }
  });
}

const eliminarAlumnoPorId = async (id) => {
  try {
    await axios.delete(`${API}/alumnos/eliminar-alumnos/${id}`);
    await cargarAlumnos();
  } catch (errr) {
    Swal.fire({ icon: 'error', title: 'Error al eliminar el alumno', text: 'No se pudo eliminar el alumno.' });
  }
}

onMounted(cargarAlumnos);
</script>