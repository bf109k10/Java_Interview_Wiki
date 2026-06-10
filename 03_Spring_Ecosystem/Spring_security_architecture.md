# Архитектура Spring Security

Архитектура Spring Security: Фильтры, JWT E2E
Spring Security работает отдельно от основного контекста Spring MVC (контроллеров) на уровне сервлетных фильтров.1. Архитектура на фильтрах и связь со Spring ContextSpring Security — это не про АОП-прокси (для веб-слоя) и не про контроллеры. Это цепочка обычных Сервлетных Фильтров (Servlet Filters), которые перехватывают HTTP-запрос ДО того, как он попадет в DispatcherServlet.Как Сервлетный контейнер связывается со Spring Context?Контейнер сервлетов (например, Tomcat) ничего не знает про бины Spring.При старте Spring регистрирует в Tomcat стандартный фильтр-мост — DelegatingFilterProxy.Когда приходит запрос, DelegatingFilterProxy перенаправляет его внутрь контекста Spring к бину FilterChainProxy.FilterChainProxy содержит список цепочек фильтров (SecurityFilterChain), сопоставляет URL запроса и запускает внутренние security-фильтры.Схема прохождения запроса с JWT:HTTP Request (в заголовке Authorization: Bearer <token>)
     ⬇️
[Tomcat / Netty Standard Filters]
     ⬇️
[DelegatingFilterProxy] (Мост между сервлетом и Spring)
     ⬇️
[FilterChainProxy (SecurityFilterChain)] 
   ├─ 1. BearerTokenAuthenticationFilter / Кастомный JwtFilter (Парсит и валидирует токен)
   ├─ 2. SecurityContextPersistenceFilter (Помещает авторизованного юзера в контекст потока)
   └─ 3. FilterSecurityInterceptor (Проверяет права доступа / Роли перед вызовом REST)
     ⬇️
[DispatcherServlet] (Начало вашего Spring MVC кода)
     ⬇️
[@RestController]
2. Ключевые компоненты: Кто за что отвечаетИнтервьюер может попросить описать процесс аутентификации по шагам. Вот главные актеры:SecurityContextHolder: Главное хранилище. По сути, это обертка над ThreadLocal, которая хранит данные о текущем вошедшем пользователе для текущего потока выполнения.Authentication: Объект внутри контекста. Содержит:Principal: Кто этот пользователь (обычно объект, реализующий UserDetails).Credentials: Пароль или токен (обычно зануляется после успешного входа ради безопасности).Authorities: Права и роли пользователя (GrantedAuthority).AuthenticationManager: Интерфейс-оркестратор. У него один метод authenticate(). Он сам не проверяет пароли, он просто по очереди опрашивает доступные провайдеры.AuthenticationProvider: Реальный исполнитель логики. Например, DaoAuthenticationProvider берет логин, идет в базу данных через UserDetailsService, достает хэш пароля и сравнивает его с помощью PasswordEncoder.3. Разница между Аутентификацией и АвторизациейАутентификация (Authentication): Ответ на вопрос "Кто ты такой?". Процесс проверки подлинности пользователя (ввод логина/пароля, проверка токена).Авторизация (Authorization): Ответ на вопрос "Что тебе разрешено делать?". Процесс проверки прав доступа (ролей) к конкретному URL или методу после того, как личность пользователя уже установлена.4. End-to-End (E2E) жизненный цикл JWT: Полный путь токеновВ современных REST API используется схема с двумя токенами: Access Token (короткоживущий, 15 минут) и Refresh Token (долгоживущий, 7–30 дней). Это нужно, чтобы не заставлять пользователя вводить пароль каждые 15 минут, но при этом иметь возможность быстро отозвать доступ.Сквозной сценарий взаимодействия (E2E Flow):Аутентификация (Запрос): Клиент отправляет POST-запрос на /auth/login с логином и паролем в JSON.Генерация (Сервер): AuthenticationManager проверяет данные. Если всё ок, сервер генерирует пару Access и Refresh токенов. Информация о Refresh-токене сохраняется в базу данных или Redis для последующего контроля сессий.Ответ (Сервер -> Клиент):Access Token возвращается в теле ответа (JSON). Клиент сохраняет его исключительно в оперативной памяти (In-Memory) фронтенд-приложения.Refresh Token упаковывается в защищенную куку (Set-Cookie: token=...; HttpOnly; Secure; SameSite=Strict) и отправляется клиенту. Это защищает Refresh-токен от кражи через XSS-атаки.Обычный запрос к API: Клиент прикрепляет Access Token в заголовок Authorization: Bearer <token>. Кастомный JwtAuthenticationFilter перехватывает запрос, парсит токен с помощью секретного ключа, проверяет срок его действия (exp), извлекает роли и заселяет их в SecurityContextHolder. Запрос успешно идет в @RestController.Истечение токена: Спустя 15 минут Access Token протухает. При следующем запросе сервер возвращает статус 401 Unauthorized.Обновление и Ротация (Refresh Token Rotation): Клиент перехватывает ошибку 401 и автоматически отправляет POST-запрос на /auth/refresh, прикрепляя куку с Refresh-токеном.Сервер валидирует Refresh-токен и проверяет в БД, не отозван ли он.Сервер удаляет старый Refresh-токен из базы, генерирует абсолючно новую пару (Access + Refresh) и перезаписывает куку клиенту.Если злоумышленник попытается использовать старый Refresh-токен повторно, сервер обнаружит это (Reuse Detection), поймет, что произошел взлом, и мгновенно аннулирует все сессии данного пользователя.5. Код реализации E2E-процесса в Spring SecurityКонтроллер аутентификации и обновления (AuthRestController)java@RestController
@RequestMapping("/api/v1/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthService authService;

    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@RequestBody LoginRequest request, HttpServletResponse response) {
        JwtPair jwtPair = authService.authenticate(request.getUsername(), request.getPassword());
        
        ResponseCookie cookie = ResponseCookie.from("refreshToken", jwtPair.getRefreshToken())
                .httpOnly(true)
                .secure(true) 
                .path("/api/v1/auth/refresh")
                .maxAge(7 * 24 * 60 * 60) 
                .sameSite("Strict")
                .build();
        response.addHeader(HttpHeaders.SET_COOKIE, cookie.toString());

        return ResponseEntity.ok(new AuthResponse(jwtPair.getAccessToken()));
    }

    @PostMapping("/refresh")
    public ResponseEntity<AuthResponse> refresh(@CookieValue("refreshToken") String refreshToken, HttpServletResponse response) {
        JwtPair newPair = authService.refreshSession(refreshToken);
        
        ResponseCookie cookie = ResponseCookie.from("refreshToken", newPair.getRefreshToken())
                .httpOnly(true)
                .secure(true)
                .path("/api/v1/auth/refresh")
                .maxAge(7 * 24 * 60 * 60)
                .sameSite("Strict")
                .build();
        response.addHeader(HttpHeaders.SET_COOKIE, cookie.toString());
        
        return ResponseEntity.ok(new AuthResponse(newPair.getAccessToken()));
    }
}

6. База ответов на типовые вопросы собеседований по Spring SecurityВ чем разница между Role и Authority в Spring Security?Ответ: Технически обе концепции реализуют один интерфейс — GrantedAuthority. Разница заключается исключительно в семантике и соглашении об именовании:Authority (Право) — это точечное действие, атомарное разрешение на операцию (например, READ_PROFILE, DELETE_USER).Role (Роль) — это высокоуровневая группа прав (например, ADMIN, USER). В Spring Security строка роли обязана начинаться с префикса ROLE_ (например, ROLE_ADMIN). При использовании метода .hasRole("ADMIN") Spring под капотом автоматически проверяет наличие authority с именем ROLE_ADMIN. Если префикс отсутствует, метод работать не будет.Что такое CSRF-атака и почему в REST API с JWT её обычно отключают?Ответ: CSRF (Cross-Site Request Forgery) — атака, при которой злоумышленник заставляет браузер жертвы отправить скрытый запрос на защищенный сайт, используя тот факт, что браузер автоматически прикрепляет к запросу куки сессии (Session Cookie).В классических MVC-приложениях, использующих сессии, CSRF-защита обязательна (сервер генерирует специальный токен и проверяет его в каждом POST/PUT запросе).В REST API, работающих на JWT в режиме Stateless, сервер не хранит сессии, а Access Token передается клиентом вручную в заголовке Authorization: Bearer <token>. Браузер никогда не прикрепляет заголовки автоматически (в отличие от кук). Поэтому CSRF-атака становится невозможной, и защиту отключают через http.csrf(AbstractHttpConfigurer::disable).Нюанс для Senior: Если вы храните JWT (даже Refresh-токен) в куках, уязвимость к CSRF возвращается! В таком случае куки нужно защищать флагом SameSite=Strict или настраивать CsrfTokenRepository.Как защитить конкретный метод в коде на уровне бизнес-логики (Method Security)?Ответ: Для защиты методов на уровне сервисов используется аннотация @PreAuthorize (включается с помощью @EnableMethodSecurity над конфигурационным классом). Она работает через АОП-прокси бизнес-бинов и поддерживает язык выражений SpEL (Spring Expression Language):java// Доступ только для пользователей с ролью ADMIN
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long id) { ... }

// Проверка динамического условия: текущий пользователь может менять только свой профиль
@PreAuthorize("#username == authentication.principal.username or hasRole('ADMIN')")
public void updateProfile(String username, ProfileDto dto) { ... }
Как работает шифрование паролей в Spring Security и можно ли использовать MD5?Ответ: Использовать MD5 или SHA-256 в чистом виде категорически нельзя, так как они уязвимы к атакам по радужным таблицам из-за высокой скорости хэширования.Spring Security использует интерфейс PasswordEncoder, а стандартом де-факто является BCryptPasswordEncoder.BCrypt под капотом: Он автоматически генерирует случайную соль (salt) для каждого пароля и внедряет её прямо в результирующую строку хэша. Кроме того, он использует адаптивный алгоритм шифрования с параметром strength (work factor) — это количество итераций хэширования. Это намеренно замедляет процесс проверки, делая брутфорс (подбор) паролей вычислительно невозможным для злоумышленника.Как извлечь данные текущего авторизованного пользователя в любой точке кода?Ответ: Так как Spring Security по умолчанию привязывает контекст безопасности к текущему потоку выполнения (ThreadLocal), получить данные можно через статический вызов:javaAuthentication auth = SecurityContextHolder.getContext().getAuthentication();
if (auth != null && auth.isAuthenticated()) {
    String username = auth.getName();
    MyUserDetails userDetails = (MyUserDetails) auth.getPrincipal();
}
В контроллерах Spring MVC предоставляет более элегантный способ — внедрение через аннотацию @AuthenticationPrincipal:java@GetMapping("/me")
public ResponseEntity<UserDto> getMe(@AuthenticationPrincipal MyUserDetails userDetails) {
    return ResponseEntity.ok(new UserDto(userDetails.getUsername()));
}
