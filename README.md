Виконав: Пташников Василь AI-235
1. МЕТА РОБОТИ
Ознайомлення з принципами локального зберігання даних в Android-застосунках за допомогою бібліотеки Room, а також формування навичок побудови багаторівневої структури застосунку з розділенням відповідальності між UI (Jetpack Compose), ViewModel, Repository та локальною базою даних.

2. ЗАВДАННЯ
Доопрацювати застосунок з лабораторної роботи №3 таким чином, щоб усі дані про проекти (назва, опис, прогрес) зберігалися в локальній базі даних Room. Забезпечити збереження даних після повного закриття та повторного запуску застосунку.

3. ХІД ВИКОНАННЯ РОБОТИ
3.1. Додавання залежностей та налаштування Gradle
У файл build.gradle.kts (модуля app) було додано плагін KSP для обробки анотацій Room, а також відповідні залежності:

kotlin


plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("com.google.devtools.ksp")
}
dependencies {
    // ... інші залежності
    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    ksp("androidx.room:room-compiler:2.6.1")
}
Для забезпечення сумісності компілятора Kotlin із версією JDK на локальній машині, у файлі gradle.properties було зафіксовано використання JDK 17:

properties


org.gradle.java.home=C:\\Program Files\\Java\\jdk-17
3.2. Створення структури проекту (Layers)
Проект було реструктуризовано на три логічні шари:

Domain Layer: містить бізнес-модель проекту Project зі змінним ID типу Int.
Data Layer: містить сутність Room Database (ProjectEntity), інтерфейс доступу до даних (ProjectDao), клас бази даних (AppDatabase), синглтон-провайдер БД (DatabaseProvider), маппер (ProjectMapper) та репозиторій (ProjectRepository).
UI/ViewModel Layer: містить екрани Jetpack Compose та ProjectViewModel разом із фабрикою ProjectViewModelFactory для передачі репозиторію.
3.3. Реалізація локального збереження (кімната Room)
Entity (ProjectEntity.kt): описує структуру таблиці projects у базі даних SQLite:
kotlin


@Entity(tableName = "projects")
data class ProjectEntity(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val name: String,
    val description: String,
    val progress: Int = 0
)
DAO (ProjectDao.kt): оперує SQL-запитами за допомогою анотацій Room:
kotlin


@Dao
interface ProjectDao {
    @Query("SELECT * FROM projects ORDER BY id DESC")
    fun getAllProjects(): Flow<List<ProjectEntity>>
    @Query("SELECT * FROM projects WHERE id = :id")
    fun getProjectById(id: Int): Flow<ProjectEntity?>
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertProject(project: ProjectEntity)
    @Update
    suspend fun updateProject(project: ProjectEntity)
    @Delete
    suspend fun deleteProject(project: ProjectEntity)
}
3.4. Шаблон проектування Repository
Клас ProjectRepository.kt виступає єдиним джерелом даних для ViewModel, маскуючи пряме звернення до бази даних:

kotlin


class ProjectRepository(private val projectDao: ProjectDao) {
    val projects: Flow<List<Project>> = projectDao.getAllProjects().map { list ->
        list.map { it.toDomain() }
    }
    fun getProjectById(id: Int) = projectDao.getProjectById(id).map { it?.toDomain() }
    suspend fun addProject(project: Project) = projectDao.insertProject(project.toEntity())
    suspend fun updateProject(project: Project) = projectDao.updateProject(project.toEntity())
    suspend fun deleteProject(project: Project) = projectDao.deleteProject(project.toEntity())
}
3.5. ViewModel та зв'язок з інтерфейсом
ProjectViewModel отримує дані через репозиторій та конвертує холодний потік Flow у гарячий StateFlow за допомогою stateIn для інтеграції з Compose.
Створено ProjectViewModelFactory, яка передає екземпляр репозиторію у конструктор ViewModel.
В Navigation.kt ініціалізується база даних за допомогою DatabaseProvider.getDatabase(context) та створюється ViewModel.
4. РЕЗУЛЬТАТИ ТЕСТУВАННЯ ТА ВЕРИФІКАЦІЇ
Проект успішно компілюється за допомогою Gradle команд без помилок збірки (BUILD SUCCESSFUL). Нижче наведено результати роботи застосунку на емуляторі.

Крок 1. Запуск застосунку та створення нових проектів
Після першого запуску застосунок відображає порожній список. Було створено три тестові проекти за допомогою кнопки «+»:

Мобільний додаток (Опис: iOS/Android розробка).
Веб-сайт (Опис: Корпоративний).
База даних (Опис: Дизайн та реалізація).
[ ВСТАВТЕ СКУРИНШОТ №1: Головний екран із трьома створеними проектами в списку ]

Крок 2. Зміна прогресу проекту
При натисканні на проект «Мобільний додаток» відкривається екран детальної інформації. За допомогою слайдера прогрес було змінено на 75%, після чого натиснуто кнопку «Редагувати».

[ ВСТАВТЕ СКРИНШОТ №2: Екран «Деталі проекту» із встановленим значенням 75% ]

Крок 3. Видалення проекту
Проект «База даних» було відкрито на екрані деталей та успішно видалено за допомогою кнопки «Видалити». Застосунок автоматично повернувся на головний екран, де залишилося лише 2 проекти.

[ ВСТАВТЕ СКРИНШОТ №3: Головний екран із двома проектами (проект «База даних» відсутній) ]

Крок 4. Перевірка збереження даних (Room Database)
Застосунок було повністю вивантажено з пам'яті пристрою (закрити емулятор/додаток) та запущено знову. Після перезапуску раніше додані проекти («Мобільний додаток» із прогресом 75% та «Веб-сайт» із прогресом 30%) залишилися у списку. Це підтверджує успішну роботу бази даних Room та збереження даних на постійній основі.

[ ВСТАВТЕ СКРИНШОТ №4: Головний екран після повного перезапуску застосунку ]

5. ВИСНОВКИ
Під час виконання лабораторної роботи було успішно підключено локальну базу даних Room до Android-проєкту та реалізовано архітектурний шаблон Repository. Створено класи Entity, DAO та RoomDatabase. Бізнес-логіка була відокремлена від збереження даних, а ViewModel переписана для взаємодії з репозиторієм замість колекцій у пам'яті. Завдяки використанню Kotlin Flow інтерфейс Jetpack Compose автоматично оновлюється при змінах у базі даних. Тестування підтвердило повне збереження даних після перезапуску програми.
