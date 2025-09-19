Nota: El codigo funciona pero no esta realizando la accion que se requiere.

Codigo de comboBox, para asignar un usuario a un tarea.

```js
Filter(

    DBTUsuario,

    // Usuarios que pertenecen al proyecto actual

    ID in ForAll(

        Filter(DBTProyectoUsuarios, idProyecto.Id = varProyectoActual),

        idUsuario.Id

    )

    &&

    // Que no estén ya asignados en DBTTareaUsuario dentro de este proyecto

    !(ID in ForAll(

        Filter(

            DBTTareaUsuario,

            idTarea.Id in ForAll(

                Filter(DBTTareas, idProyecto.Id = varProyectoActual),

                ID

            )

        ),

        idUsuario.Id

    ))

)
```

## Asignar una tarea un Usuario

```js
If(
    // 1. Validar selección en los combos
    IsBlank(cmbUserAsignarTarea.Selected) || IsBlank(cmbUserRol.Selected),
    Notify("Debes seleccionar un usuario y un rol.", NotificationType.Error),

    // 2. Buscar si ya existe asignación de ese usuario en la tarea
    With(
        {
            tareaAsignada: LookUp(
                DBTTareaUsuario,
                idTarea.Id = varTareaActual.ID &&
                idUsuario.Id = cmbUserAsignarTarea.Selected.ID   // <- este ID es de DBTProyectoUsuarios
            )
        },
        If(
            !IsBlank(tareaAsignada),
            // 👉 Actualizar asignación existente
            Patch(
                DBTTareaUsuario,
                tareaAsignada,
                {
                    rol: { Value: cmbUserRol.Selected.Value },
                    fechaAsignacion: Now()
                }
            ),
            // 👉 Crear nueva asignación
            Patch(
                DBTTareaUsuario,
                Defaults(DBTTareaUsuario),
                {
                    idTarea: { Id: varTareaActual.ID },
                    idUsuario: { Id: cmbUserAsignarTarea.Selected.ID },  // referencia al registro en DBTProyectoUsuarios
                    rol: { Value: cmbUserRol.Selected.Value },
                    fechaAsignacion: Now()
                }
            )
        )
    );

    // 3. Confirmación y limpieza
    Notify("Usuario asignado a la tarea.", NotificationType.Success);
    Reset(cmbUserAsignarTarea);
    Reset(cmbUserRol)
)

```

## Estados del boton de la tarea
```js
If(

    varTareaActual.estadoTarea.Value = "Inicial",

    // Avanzar a En Progreso y actualizar varTareaActual con el Patch

    Set(

        varTareaActual,

        Patch(

            DBTTareas,

            varTareaActual,

            { estadoTarea: { Value: "En Progreso" } }

        )

    ),

  

    varTareaActual.estadoTarea.Value = "En Progreso",

    // Avanzar a Finalizado

    Set(

        varTareaActual,

        Patch(

            DBTTareas,

            varTareaActual,

            { estadoTarea: { Value: "Finalizado" } }

        )

    ),

  

    // Si ya es Finalizado

    varTareaActual.estadoTarea.Value = "Finalizado",

    Notify("La tarea ya está finalizada", NotificationType.Information)

)
```


## Cambiar el color de los estados del proyecto

```js
// Definir "función" de recálculo de estatus proyecto

Set(

    fnRecalcularEstatus,

    {

        Run:

            // Aquí metemos la lógica que estaba en BtnRecalcularEstatus.OnSelect

            Collect(

                colDummy,   // ⚠️ Truco: usamos un colector temporal vacío

                {

                    _exec:

                        // 1. Obtener todas las tareas asociadas al proyecto actual

                        ClearCollect(

                            colTareasProyecto,

                            Filter(

                                DBTTareas,

                                idProyecto.Id = varProyectoActual

                            )

                        );

  

                        // 2. Determinar el nuevo estado

                        Set(

                            varNuevoEstado,

                            If(

                                CountRows(colTareasProyecto) = 0,

                                "Inicial",

  

                                And(

                                    LookUp(DBTProyectos, ID = varProyectoActual).fechaVencimiento < Now(),

                                    CountRows(Filter(colTareasProyecto, estadoTarea.Value <> "Finalizado")) > 0

                                ),

                                "Sin Finalizar",

  

                                CountRows(Filter(colTareasProyecto, estadoTarea.Value = "Finalizado")) = CountRows(colTareasProyecto),

                                "Finalizado",

  

                                "En Progreso"

                            )

                        );

  

                        // 3. Actualizar el estado en DBTProyectos

                        Patch(

                            DBTProyectos,

                            LookUp(DBTProyectos, ID = varProyectoActual),

                            {

                                estado: LookUp(

                                    Choices(DBTProyectos.estado),

                                    Value = varNuevoEstado

                                )

                            }

                        );

  

                        // 4. Refrescar el proyecto actual

                        Set(

                            varProyectoActualRecord,

                            LookUp(DBTProyectos, ID = varProyectoActual)

                        )

                }

            )

    }

);
```

## Filtrar por fecha

```js
//DBTProyectos

Filter(

    DBTProyectos,

    // ✅ Filtro por visibilidad

    (

        filtrarProyectoPrivate.Selected.Value = "Todos" ||

        (filtrarProyectoPrivate.Selected.Value = "Público" && esPublico = true) ||

        (filtrarProyectoPrivate.Selected.Value = "Privado" && esPublico = false)

    )

    &&

    // ✅ Filtro por estado

    (

        filtrarProyectoEstado.Selected.Value = "Todos" ||

        estado.Value = filtrarProyectoEstado.Selected.Value

    )

    &&

    // ✅ Filtro por fecha

    (

        cmbDia.Selected.Value = "Todos" ||

        (cmbDia.Selected.Value = "Hoy" && DateValue(fechaCreacion) = Today()) ||

       (cmbDia.Selected.Value = "Esta Semana" && WeekNum(DateValue(fechaCreacion)) = WeekNum(Today()) && Year(DateValue(fechaCreacion)) = Year(Today())) ||

        (cmbDia.Selected.Value = "Este Mes" && Month(DateValue(fechaCreacion)) = Month(Today()) && Year(DateValue(fechaCreacion)) = Year(Today()))

    ) &&

    // ✅ Filtro por día específico (ComboBox)

    (

        IsBlank(cmbDia.Selected.Value) || cmbDia.Selected.Value = "Todos" ||

        Day(DateValue(fechaCreacion)) = Value(cmbDia.Selected.Value)

    )

    &&

    // ✅ Filtro por mes específico (ComboBox)

    (

        IsBlank(cmbMes.Selected.Value) || cmbMes.Selected.Value = "Todos" ||

        Month(DateValue(fechaCreacion)) = Value(cmbMes.Selected.Value)

    )

    &&

    // ✅ Filtro por año específico (ComboBox)

    (

        IsBlank(cmbAnio.Selected.Value) || cmbAnio.Selected.Value = "Todos" ||

        Year(DateValue(fechaCreacion)) = Value(cmbAnio.Selected.Value)

    )

  

)
```

## Configuracion de ComboBox filtro de Actividad

```js
    (
        filtrarProyectoDia1.Selected.Value = "Todos" ||

        (filtrarProyectoDia1.Selected.Value = "Hoy" && DateValue(fechaCreacion) = Today()) ||

        (filtrarProyectoDia1.Selected.Value = "Esta Semana" && WeekNum(DateValue(fechaCreacion)) = WeekNum(Today()) && Year(DateValue(fechaCreacion)) = Year(Today())) ||

        (filtrarProyectoDia1.Selected.Value = "Este Mes" && Month(DateValue(fechaCreacion)) = Month(Today()) && Year(DateValue(fechaCreacion)) = Year(Today()))

    )
```

## Listar los proyectos de usuarios

```js
// 1) obtener el registro y ID del usuario actual

Set(varUsuario, LookUp(DBTUsuario, email = User().Email));

Set(varUsuarioID, If(IsBlank(varUsuario), Blank(), varUsuario.ID));

  

// 2) limpiar colección

Clear(colMisProyectos);

  

// 3) agregar proyectos que creó (Rol = "Creador")

ForAll(

    Filter(DBTProyectos, idCreador.Id = varUsuarioID),

    Collect(

        colMisProyectos,

        {

            ID: ID,

            nombreProyecto: nombreProyecto,

            fechaVencimiento: fechaVencimiento,

            idCreador: idCreador,

            Rol: "Creador"

        }

    )

);

  

// 4) agregar proyectos donde está asignado (Rol = "Asignado"), sin duplicados

ForAll(

    Filter(DBTProyectoUsuarios, idUsuario.Id = varUsuarioID),

    With(

        { proj: LookUp(DBTProyectos, ID = idProyecto.Id) },

        If(

            !IsBlank(proj) && IsBlank(LookUp(colMisProyectos, ID = proj.ID)),

            Collect(

                colMisProyectos,

                {

                    ID: proj.ID,

                    nombreProyecto: proj.nombreProyecto,

                    fechaVencimiento: proj.fechaVencimiento,

                    idCreador: proj.idCreador,

                    Rol: "Asignado"

                }

            )

        )

    )

);
```

## Coleccion de Tareas

```js
// Colección de Tareas

// 1) Limpiar colecciones
Clear(colMisTareas);
ClearCollect(
    colTareasAsignadas,
    ForAll(
        Filter(DBTTareaUsuario, idUsuario.Id = varUsuarioID),
        { TareaID: idTarea.Id }
    )
);

// 2) Tareas creadas por el usuario (Rol = "Creador")
ForAll(
    Filter(DBTTareas, idCreador.Id = varUsuarioID),
    Collect(
        colMisTareas,
        {
            ID: ID,
            nombreTarea: nombreTarea,
            descripcion: descripcion,
            fechaVencimiento: fechaVencimiento,   // 📅 Fecha límite de la tarea
            idProyecto: idProyecto,
            idCreador: idCreador,
            Rol: "Creador",
            Estado: If(IsBlank(estadoTarea), "", estadoTarea.Value),   // Choice
            Prioridad: If(IsBlank(prioridad), "", prioridad.Value),    // Choice

            // 🔥 NUEVOS CAMPOS
            nombreProyecto: LookUp(DBTProyectos, ID = idProyecto.Id, nombreProyecto),
            DiasRestantes: If(
                IsBlank(fechaVencimiento),
                Blank(),
                DateDiff(Today(), fechaVencimiento, Days)
            ),
            ColorSemaforo: If(
                IsBlank(fechaVencimiento), Gray,
                fechaVencimiento < Today(), Red,
                DateDiff(Today(), fechaVencimiento, Days) <= 3, Orange,
                Green
            )
        }
    )
);

// 3) Tareas donde está asignado (Rol = "Asignado") sin duplicados
ForAll(
    Filter(DBTTareaUsuario, idUsuario.Id = varUsuarioID),
    With(
        { tarea: LookUp(DBTTareas, ID = idTarea.Id) },
        If(
            !IsBlank(tarea) && IsBlank(LookUp(colMisTareas, ID = tarea.ID)),
            Collect(
                colMisTareas,
                {
                    ID: tarea.ID,
                    nombreTarea: tarea.nombreTarea,
                    descripcion: tarea.descripcion,
                    fechaVencimiento: tarea.fechaVencimiento,   // 📅 Fecha límite de la tarea
                    idProyecto: tarea.idProyecto,
                    idCreador: tarea.idCreador,
                    Rol: "Asignado",
                    Estado: If(IsBlank(tarea.estadoTarea), "", tarea.estadoTarea.Value), // Choice
                    Prioridad: If(IsBlank(tarea.prioridad), "", tarea.prioridad.Value),   // Choice

                    // 🔥 NUEVOS CAMPOS
                    nombreProyecto: LookUp(DBTProyectos, ID = tarea.idProyecto.Id, nombreProyecto),
                    DiasRestantes: If(
                        IsBlank(tarea.fechaVencimiento),
                        Blank(),
                        DateDiff(Today(), tarea.fechaVencimiento, Days)
                    ),
                    ColorSemaforo: If(
                        IsBlank(tarea.fechaVencimiento), Gray,
                        tarea.fechaVencimiento < Today(), Red,
                        DateDiff(Today(), tarea.fechaVencimiento, Days) <= 3, Orange,
                        Green
                    )
                }
            )
        )
    )
);

// 4) (Opcional) Ordenar la colección por fecha de vencimiento ascendente
ClearCollect(
    colMisTareas,
    SortByColumns(colMisTareas, "fechaVencimiento", Ascending)
);
```

## Primera configuración
```js
  
  

// Coleccion de Tareas

// 2) Limpiar colecciones

Clear(colMisTareas);

ClearCollect(

    colTareasAsignadas,

    ForAll(

        Filter(DBTTareaUsuario, idUsuario.Id = varUsuarioID),

        { TareaID: idTarea.Id }

    )

);

  

// 3) Tareas creadas por el usuario (Rol = "Creador")

ForAll(

    Filter(DBTTareas, idCreador.Id = varUsuarioID),

    Collect(

        colMisTareas,

        {

            ID: ID,

            nombreTarea: nombreTarea,

            descripcion: descripcion,

            fechaVencimiento: fechaVencimiento,   // 📅 Fecha límite de la tarea

            idProyecto: idProyecto,               // referencia al proyecto

            idCreador: idCreador,

            Rol: "Creador",

            Estado: If(IsBlank(estadoTarea), "", estadoTarea.Value),   // <-- Choice

            Prioridad: If(IsBlank(prioridad), "", prioridad.Value)     // <-- Choice

        }

    )

);

  

// 4) Tareas donde está asignado (Rol = "Asignado") sin duplicados

ForAll(

    Filter(DBTTareaUsuario, idUsuario.Id = varUsuarioID),

    With(

        { tarea: LookUp(DBTTareas, ID = idTarea.Id) },

        If(

            !IsBlank(tarea) && IsBlank(LookUp(colMisTareas, ID = tarea.ID)),

            Collect(

                colMisTareas,

                {

                    ID: tarea.ID,

                    nombreTarea: tarea.nombreTarea,

                    descripcion: tarea.descripcion,

                    fechaVencimiento: tarea.fechaVencimiento,   // 📅 Fecha límite de la tarea

                    idProyecto: tarea.idProyecto,

                    idCreador: tarea.idCreador,

                    Rol: "Asignado",

                    Estado: If(IsBlank(tarea.estadoTarea), "", tarea.estadoTarea.Value), // <-- Choice

                    Prioridad: If(IsBlank(tarea.prioridad), "", tarea.prioridad.Value)   // <-- Choice

                }

            )

        )

    )

);
```


## Filtro para Proyectos de todos los proyectos

```js
Filter(

    DBTProyectos,

    // filtro por visibilidad

    (

        filtrarProyectoPrivate.Selected.Value = "Todos" ||

        (filtrarProyectoPrivate.Selected.Value = "Público" && esPublico = true) ||

        (filtrarProyectoPrivate.Selected.Value = "Privado" && esPublico = false)

    )

    &&

    // filtro por estado

    (

        filtrarProyectoEstado.Selected.Value = "Todos" ||

        estado.Value = filtrarProyectoEstado.Selected.Value

    )

    &&

    // filtro por fecha predefinida

    (

        filtrarProyectoDia.Selected.Value = "Todos" ||

        (filtrarProyectoDia.Selected.Value = "Hoy" && DateValue(fechaCreacion) = Today()) ||

        (filtrarProyectoDia.Selected.Value = "Esta Semana" && WeekNum(DateValue(fechaCreacion)) = WeekNum(Today()) && Year(DateValue(fechaCreacion)) = Year(Today())) ||

        (filtrarProyectoDia.Selected.Value = "Este Mes" && Month(DateValue(fechaCreacion)) = Month(Today()) && Year(DateValue(fechaCreacion)) = Year(Today()))

    )

  
  

    &&

    // filtro por mes específico

    (

        IsBlank(cmbMes.Selected.Value) ||

        cmbMes.Selected.Value = "Todos" ||

        Month(DateValue(fechaCreacion)) = Value(cmbMes.Selected.Value)

    )

    &&

    // filtro por día específico

    (

        IsBlank(cmbDia.Selected.Value) ||

        cmbDia.Selected.Value = "Todos" ||

        Day(DateValue(fechaCreacion)) = Value(cmbDia.Selected.Value)

    )

    &&

    // filtro por año específico

    (

        IsBlank(cmbAnio.Selected.Value) ||

        cmbAnio.Selected.Value = "Todos" ||

        Year(DateValue(fechaCreacion)) = Value(cmbAnio.Selected.Value)

    )

)
```


## Crear proyecto | Funcional pero muestra todos los proyectos a todos los usuarios

```js
// Obtener el usuario actual

ClearCollect(

    colUsuarioActual,

    Filter(DBTUsuario, Lower(Trim(email)) = Lower(Trim(User().Email)))

);

  

Set(varUsuario, First(colUsuarioActual));

  
  

// Validar si existe el usuario

If(

    IsBlank(varUsuario),

    Notify(

        "Tu usuario no está registrado en la base de datos.",

        NotificationType.Error

    ),

  

    // Validar que el nombre del proyecto no esté vacío

    If(

        IsBlank(txtNombreProyecto.Text),

        Notify(

            "Debes ingresar un nombre para el proyecto.",

            NotificationType.Error

        ),

  

        // Si usuario existe y nombre no está vacío, evaluamos modo

        If(

            varModoFormulario = "Nuevo",

            // -------------------- CREAR PROYECTO --------------------

            Set(

                varNuevoProyecto,

                Patch(

                    DBTProyectos,

                    Defaults(DBTProyectos),

                    {

                        nombreProyecto: txtNombreProyecto.Text,

                        descripcion: txtDescripcion.Text,

                        esPublico: tglEsPublico.Value,

                        fechaVencimiento: dpFechaVencimiento.SelectedDate,

                        idCreador: {

                            '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedReference",

                            Id: varUsuario.ID,

                            Value: varUsuario.email

                        },

                        fechaCreacion: Now(),

                        estado: {

                            '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedReference",

                            Value: "Inicial"

                        }

                    }

                )

            );

            Notify(

                "Proyecto creado correctamente",

                NotificationType.Success

            ),

  

            // -------------------- EDITAR PROYECTO --------------------

            Patch(

                DBTProyectos,

                varProyectoSeleccionado,

                {

                    nombreProyecto: txtNombreProyecto.Text,

                    descripcion: txtDescripcion.Text,

                    esPublico: tglEsPublico.Value,

                    fechaVencimiento: dpFechaVencimiento.SelectedDate

                    // No editamos idCreador, fechaCreacion ni estado desde aquí

                }

            );

            Notify(

                "Proyecto actualizado correctamente",

                NotificationType.Success

            )

        );

  

        // -------------------- LIMPIAR Y NAVEGAR --------------------

        Reset(txtNombreProyecto);

        Reset(txtDescripcion);

        Reset(dpFechaVencimiento);

        Navigate(Principal, ScreenTransition.Fade)

    )

)
```

## Crear Tarea | del Formulario Crear Tarea

```js
// Paso 1: Obtener el usuario actual

ClearCollect(

    colUsuarioActual,

    Filter(DBTUsuario, Lower(Trim(email)) = Lower(Trim(User().Email)))

);

  

Set(varUsuario, First(colUsuarioActual));

  
  
  

// Validar que el usuario está registrado

If(

    IsBlank(varUsuario),

    Notify(

        "Tu usuario no está registrado en la base de datos.",

        NotificationType.Error

    ),

  

    // Validar que el nombre de la tarea no esté vacío

    If(

        IsBlank(txtNombreTarea.Text),

        Notify(

            "Debes agregar un nombre a la tarea.",

            NotificationType.Error

        ),

  

        // Guardar tarea según el modo

        If(

            varModoFormulario = "Editar",

            // -------------------- EDITAR --------------------

            Patch(

                DBTTareas,

                varTareaSeleccionada,

                {

                    nombreTarea: txtNombreTarea.Text,

                    descripcion: txtDescripcion1.Text,

                    estadoTarea: { Value: cmbEstado.Selected.Value },

                    prioridad: { Value: cmbPrioridad.Selected.Value },

                    fechaVencimiento: FechaVencimiento.SelectedDate

                }

            );

            Notify("Tarea editada correctamente", NotificationType.Success),

            // 🔄 Recalcular estatus proyecto

            //fnRecalcularEstatus.Run,

  

            // -------------------- CREAR --------------------

            Patch(

                DBTTareas,

                Defaults(DBTTareas),

                {

                    nombreTarea: txtNombreTarea.Text,

                    descripcion: txtDescripcion1.Text,

                    idProyecto: {

                        '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedReference",

                        Id: varProyectoActual,

                        Value: LookUp(DBTProyectos, ID = varProyectoActual).nombreProyecto

                    },

                    idCreador: {

                        '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedReference",

                        Id: varUsuario.ID,

                        Value: varUsuario.nombre

                    },

                    estadoTarea: { Value: cmbEstado.Selected.Value },

                    prioridad: { Value: cmbPrioridad.Selected.Value },

                    fechaCreacion: Now(),

                    fechaVencimiento: FechaVencimiento.SelectedDate

                }

            );

            Notify("Tarea creada correctamente", NotificationType.Success);

             // 🔄 Recalcular estatus proyecto

             varProyectoActualRecord;

        );

  

        // -------------------- RESET Y NAVEGACIÓN (común a ambos) --------------------

        Reset(txtNombreTarea);

        Reset(txtDescripcion1);

        Reset(cmbEstado);

        Reset(cmbPrioridad);

  

        Navigate(ListaTareas, ScreenTransition.Fade)

    )

)
```

## Galeria Proyectos | Muestra todos los proyecto aun que no formen parte el Usuario

```js
  

SortByColumns(

    Filter(

        DBTTareas,

        idProyecto.Id = varProyectoActual &&

        (

            filtrarTareas.Selected.Value = "Todos" ||

            estadoTarea.Value = filtrarTareas.Selected.Value

        )

    ),

    "fechaCreacion",   // nombre del campo de fecha

    varOrdenFecha

)
```

Nota: La app, tiene problemas para validar el correo de los demas usuarios, para crear proyectos, crear tareas. Filtrar sus tareas y proyectos, ya se creador o asignado.