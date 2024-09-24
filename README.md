# Documentação do Projeto AppDemoT4Face

## Arquitetura Geral

O projeto segue uma arquitetura **MVVM (Model-View-ViewModel)**, utilizando a tecnologia  **Kotlin** junto com framework **Jetpack Compose** para a construção da interface de usuário e o Retrofit para a comunicação com a API. Abaixo, estão descritas as principais camadas e seus componentes:

1. **API**: A camada de API é implementada utilizando Retrofit, que facilita a comunicação com serviços REST. O `ApiService` define os endpoints disponíveis e como os dados serão enviados e recebidos da API.
2. **Components**: `Header` e `DropdownMenu` componentes reutilizáveis 
3. **Model**: Contém as classes que representam os dados manipulados no aplicativo, como `Face` e `PersonFace`. Esses modelos são utilizados para armazenar e transportar informações sobre as faces e suas características.
4. **Screens**: Define as telas do aplicativo, como `HomeScreen` e `AddFaceScreen`. Elas utilizam os componentes do Jetpack Compose para renderizar a interface de usuário, solicitar interações e navegar entre telas.
5. **Utils**: Um utilitário para lidar com certificados SSL inseguros está presente na classe `UnsafeOkHttpClient`, o que permite a comunicação com servidores que utilizam certificados SSL auto-assinados ou não validados (uso recomendado apenas em ambiente de desenvolvimento).
6. **ViewModel**: Responsável pela lógica de negócio e comunicação com a API. O `ApiViewModel` realiza operações como buscar as faces cadastradas e enviar novas faces para o backend.
7. **Navegação**: O arquivo AppNavHost.kt define o sistema de navegação do aplicativo, utilizando o Jetpack Navigation Compose. Ele controla o fluxo entre as diferentes telas do aplicativo, como HomeScreen, AddFaceScreen e VerifyScreen, além de gerenciar o estado da navegação utilizando o NavHostController e os parâmetros de rota definidos em AppScreen.

---

### 1. API Service

O `ApiService.kt` define dois endpoints:

- `getPersonFaces`: realiza uma requisição GET para buscar a lista de faces cadastradas no servidor.
- `addPersonFace`: realiza uma requisição POST com uma imagem e um nome para cadastrar uma nova face.
- `deletePersonFace`: realiza uma requisição DELETE com um ID por `path` para deletar um face.
- `verifyPersonFace`: realiza uma requisição POST com um NAME por `query` e uma imagem para verificar uma face.

```kotlin
interface ApiService {

    @GET("/personface")
    suspend fun getPersonFaces(): List<PersonFace>

    @Multipart
    @POST("/personface/addface")
    suspend fun addPersonFace(
        @Part("name") name: RequestBody,
        @Part personFace: MultipartBody.Part
    ): retrofit2.Response<ResponseBody>

    @DELETE("/personface/{id}")
    suspend fun deletePersonFace(
        @Path("id") id: Int
    ): Response<DeleteFaceResponse>

    @Multipart
    @POST("/personface/verify")
    suspend fun verifyPersonFace(
        @Query("name") name: String,
        @Part verifyImage: MultipartBody.Part
    ): Response<ResponseBody>
}
```
### 2. Model

Os modelos são simples e diretos, representando as entidades necessárias para a aplicação:
- `Face`: representa uma face com um ID, nome e URL da imagem.
```kotlin
data class Face(val id: Int = 0, val name: String, val imageUrl: String?)
```
- `PersonFace`: utilizada para armazenar informações mais detalhadas da face cadastrada.
```kotlin
data class PersonFace(val id: Int, val name: String, val feature: String)
```
- `DeleteFaceResponse`: esse modelo representa o retorno da requisição
```kotlin
data class DeleteFaceResponse (val total: Int)
```
- `VerifyPersonFace`: representa uma face com um nome e uma imagem
```kotlin
data class VerifyPersonFace(val name: String, val image: MultipartBody.Part)
```

### 3. Screens - Interface de Usuário

#### 3.1 AddFaceScreen

A tela `AddFaceScreen` permite que o usuário capture uma imagem usando a câmera e faça o cadastro dessa face no sistema. Utiliza o `CameraX` para captura de imagens

<details>
    <summary>Mostrar código</summary>
    
```kotlin
@Composable
fun AddFaceScreen(navController: NavController, apiViewModel: ApiViewModel = ApiViewModel()) {
    var name by remember { mutableStateOf("") }
    var capturedImage by remember { mutableStateOf<Bitmap?>(null) }
    var isCameraPreviewVisible by remember { mutableStateOf(false) }
    var isCameraPermissionGranted by remember { mutableStateOf(false) }
    var isLoading by remember { mutableStateOf(false) }
    var successMessage by remember { mutableStateOf<String?>(null) }
    var errorMessage by remember { mutableStateOf<String?>(null) }

    val context = LocalContext.current

    // permission de câmera
    val cameraPermissionLauncher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.RequestPermission()
    ) { granted: Boolean ->
        isCameraPermissionGranted = granted
        if (!granted) {
            println("Permission de câmera renegade")
        }
    }

    // verifier se a permission da camera
    val cameraPermissionGranted = ContextCompat.checkSelfPermission(
        context, Manifest.permission.CAMERA
    ) == PackageManager.PERMISSION_GRANTED

    LaunchedEffect(Unit) {
        if (!cameraPermissionGranted) {
            cameraPermissionLauncher.launch(Manifest.permission.CAMERA)
        } else {
            isCameraPermissionGranted = true
        }
    }

    Scaffold(
        topBar = {
            Header(navController = navController, title = "Cadastrar Face")
        }
    ) { paddingValues ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(paddingValues)
                .padding(16.dp),
            verticalArrangement = Arrangement.Top,
            horizontalAlignment = Alignment.CenterHorizontally
        ) {
            if (isLoading) {
                CircularProgressIndicator() // loading...
            } else {
                TextField(
                    value = name,
                    onValueChange = { name = it },
                    label = { Text("Nome da Face") },
                    modifier = Modifier.fillMaxWidth()
                )

                Spacer(modifier = Modifier.height(16.dp))

                // botao para ativar a câmera e capturar a face
                if (capturedImage == null) {
                    Button(
                        onClick = {
                            if (isCameraPermissionGranted) {
                                isCameraPreviewVisible = true
                            } else {
                                cameraPermissionLauncher.launch(Manifest.permission.CAMERA)
                            }
                        },
                        modifier = Modifier.fillMaxWidth()
                    ) {
                        Text("Ativar Câmera")
                    }
                }

                Spacer(modifier = Modifier.height(16.dp))

                // exibe a camera se o preview estiver ativo
                if (isCameraPreviewVisible) {
                    CameraPreviewView(
                        modifier = Modifier
                            .fillMaxWidth()
                            .height(300.dp),
                        onImageCaptured = { bitmap ->
                            capturedImage = bitmap // captura o bitmap da câmera
                            isCameraPreviewVisible = false // fecha a visualizaçao da camera apos captura
                        }
                    )
                }

                Spacer(modifier = Modifier.height(16.dp))

                // exibe uma previa da imagem capturada
                capturedImage?.let { bitmap ->
                    Text(text = "Imagem capturada com sucesso!")
                    Spacer(modifier = Modifier.height(8.dp))
                    Image(
                        bitmap = bitmap.asImageBitmap(),
                        contentDescription = "Imagem capturada",
                        modifier = Modifier
                            .size(200.dp)
                            .padding(16.dp)
                    )

                    // botao para refazer a captura
                    Button(
                        onClick = {
                            capturedImage = null // limpa a imagem capturada para permitir uma nova captura
                            isCameraPreviewVisible = true // Volta a exibir a câmera
                        },
                        modifier = Modifier.padding(top = 8.dp)
                    ) {
                        Text("Refazer Captura")
                    }
                }

                Spacer(modifier = Modifier.height(32.dp))

                //exibe mensagem de sucesso ou erro
                successMessage?.let {
                    Text(text = it, color = MaterialTheme.colorScheme.primary)
                }

                errorMessage?.let {
                    Text(text = it, color = MaterialTheme.colorScheme.error)
                }

                // botao para cadastrar
                if (capturedImage != null && !isLoading) {
                    Button(
                        onClick = {
                            if (name.isNotEmpty()) {
                                isLoading = true
                                apiViewModel.addPersonFace(
                                    context = context,
                                    name = name,
                                    image = capturedImage!!,
                                    onSuccess = {
                                        successMessage = "Face cadastrada com sucesso!"
                                        isLoading = false
                                        navController.navigate(AppScreen.HomeScreen.route)
                                    },
                                    onError = {
                                        errorMessage = "Erro ao cadastrar face: ${it.message}"
                                        isLoading = false
                                    }
                                )
                            } else {
                                errorMessage = "Preencha todos os campos."
                            }
                        },
                        modifier = Modifier.fillMaxWidth()
                    ) {
                        Text("Cadastrar Face")
                    }
                }
            }
        }
    }
}

```
</details>

#### 3.2 HomeScreen

A tela principal exibe a lista de faces cadastradas, utilizando um `LazyColumn` para carregar os dados de forma eficiente.

<details>
    <summary>Mostrar código</summary>
    
```kotlin
@Composable
fun HomeScreen(navController: NavController, apiViewModel: ApiViewModel = ApiViewModel()) {
    var faces by remember { mutableStateOf(listOf<PersonFace>()) }
    var errorMessage by remember { mutableStateOf<String?>(null) }
    var isLoading by remember { mutableStateOf(true) } // Controle de carregamento
    var isDeleting by remember { mutableStateOf(false) } // Controle de exclusão
    var showDeleteDialog by remember { mutableStateOf(false) } // Controla o diálogo de confirmação
    var faceToDelete by remember { mutableStateOf<PersonFace?>(null) } // Face a ser deletada

    LaunchedEffect(Unit) {
        apiViewModel.fetchPersonFaces(
            onSuccess = { fetchedFaces ->
                errorMessage = null
                faces = fetchedFaces
                isLoading = false // Carregamento completo
            },
            onError = { error ->
                errorMessage = error.message
                isLoading = false // Carregamento completo mesmo com erro
            }
        )
    }

    Scaffold(
        topBar = {
            Header(navController = navController, title = "Faces")  // Usando o header reutilizável
        }
    ) { paddingValues ->

        if (isLoading) {
            Box(
                modifier = Modifier
                    .fillMaxSize()
                    .padding(paddingValues),
                contentAlignment = Alignment.Center // Centraliza o loading
            ) {
                CircularProgressIndicator() // Indicador de carregamento principal
            }
        } else {
            if (errorMessage != null) {
                Box(
                    modifier = Modifier
                        .fillMaxSize()
                        .padding(paddingValues),
                    contentAlignment = Alignment.Center
                ) {
                    Card(
                        modifier = Modifier.padding(16.dp),
                        shape = RoundedCornerShape(16.dp),
                        elevation = CardDefaults.elevatedCardElevation(8.dp),
                        colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.primary)
                    ) {
                        Text(
                            text = "Nenhuma face disponível no momento!",
                            modifier = Modifier.padding(16.dp),
                            style = MaterialTheme.typography.bodyMedium,
                            color = MaterialTheme.colorScheme.onSecondary,
                            textAlign = TextAlign.Center
                        )
                    }
                }
            } else if (faces.isEmpty()) {
                Box(
                    modifier = Modifier
                        .fillMaxSize()
                        .padding(paddingValues),
                    contentAlignment = Alignment.Center
                ) {
                    Card(
                        modifier = Modifier.padding(16.dp),
                        shape = RoundedCornerShape(16.dp),
                        elevation = CardDefaults.elevatedCardElevation(8.dp),
                        colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.primary)
                    ) {
                        Text(
                            text = "Nenhuma face disponível no momento!",
                            modifier = Modifier.padding(16.dp),
                            style = MaterialTheme.typography.bodyMedium,
                            color = MaterialTheme.colorScheme.onSecondary,
                            textAlign = TextAlign.Center
                        )
                    }
                }
            } else {
                LazyColumn(modifier = Modifier.padding(paddingValues)) {
                    items(faces) { face ->
                        PersonFaceItem(
                            personFace = face,
                            onDeleteClick = { faceToDeleteItem ->
                                // Exibe o diálogo de confirmação antes de deletar
                                faceToDelete = faceToDeleteItem
                                showDeleteDialog = true
                            }
                        )
                    }
                }
            }

            // Exibe o diálogo de confirmação
            if (showDeleteDialog) {
                AlertDialog(
                    onDismissRequest = { showDeleteDialog = false },
                    title = { Text(text = "Confirmação de Exclusão") },
                    text = { Text(text = "Tem certeza que deseja excluir esta face?") },
                    confirmButton = {
                        Button(
                            onClick = {
                                showDeleteDialog = false
                                isDeleting = true

                                faceToDelete?.let { face ->
                                    apiViewModel.deletePersonFace(
                                        id = face.id,
                                        onSuccess = {
                                            faces = faces.filter { it.id != face.id }
                                            isDeleting = false
                                        },
                                        onError = { error ->
                                            errorMessage = "Erro ao excluir face: ${error.message}"
                                            isDeleting = false
                                        }
                                    )
                                }
                            }
                        ) {
                            Text("Excluir")
                        }
                    },
                    dismissButton = {
                        Button(onClick = { showDeleteDialog = false }) {
                            Text("Cancelar")
                        }
                    }
                )
            }

            // Exibe o indicador de exclusão (loading) se estiver deletando
            if (isDeleting) {
                Box(
                    modifier = Modifier.fillMaxSize(),
                    contentAlignment = Alignment.Center
                ) {
                    CircularProgressIndicator() // Indicador de carregamento para exclusão
                }
            }
        }
    }
}

@Composable
fun PersonFaceItem(
    personFace: PersonFace,
    onDeleteClick: (PersonFace) -> Unit
) {
    // Modern Card layout
    Card(
        modifier = Modifier
            .padding(8.dp)
            .fillMaxWidth(),
        elevation = CardDefaults.elevatedCardElevation(6.dp),
        shape = RoundedCornerShape(12.dp)
    ) {
        Row(
            modifier = Modifier
                .padding(16.dp)
                .fillMaxWidth(),
            verticalAlignment = Alignment.CenterVertically
        ) {
            // Exibe as informações da face
            Column(
                modifier = Modifier.weight(1f)
            ) {
                Text(
                    text = "ID: ${personFace.id}",
                    style = MaterialTheme.typography.bodyLarge
                )
                Spacer(modifier = Modifier.height(4.dp))
                Text(
                    text = "Nome: ${personFace.name}",
                    style = MaterialTheme.typography.bodyMedium,
                    color = Color.Gray
                )
            }

            // Botão de deletar
            IconButton(
                onClick = { onDeleteClick(personFace) },
                modifier = Modifier.size(32.dp)
            ) {
                Icon(
                    imageVector = Icons.Sharp.Delete,
                    contentDescription = "Excluir face",
                    tint = MaterialTheme.colorScheme.primary,
                    modifier = Modifier.size(24.dp)
                )
            }
        }
    }
}

```
</details>

#### 3.3 VerifyScreen

Exibe um campo de entrada para o nome da face, um botão para selecionar a imagem da galeria e um botão para submeter para a api.

<details>
  <summary>Mostrar código</summary>

```kotlin
@Composable
fun VerifyScreen(navController: NavController, apiViewModel: ApiViewModel = ApiViewModel()) {
    var name by remember { mutableStateOf("") }
    var selectedImageUri by remember { mutableStateOf<Uri?>(null) }
    var selectedImageBitmap by remember { mutableStateOf<Bitmap?>(null) }
    var isLoading by remember { mutableStateOf(false) }
    var showDialog by remember { mutableStateOf(false) }
    var dialogMessage by remember { mutableStateOf<String?>(null) }

    val context = LocalContext.current

    // Lançador para selecionar imagem da galeria
    val imagePickerLauncher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.GetContent()
    ) { uri: Uri? ->
        selectedImageUri = uri
        selectedImageUri?.let {
            selectedImageBitmap = getBitmapFromUri(it, context, maxWidth = 1024, maxHeight = 1024) // Redimensiona a imagem
        }
    }

    Scaffold(
        topBar = {
            Header(navController = navController, title = "Verificar Face")
        }
    ) { paddingValues ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(paddingValues)
                .padding(16.dp),
            verticalArrangement = Arrangement.Top,
            horizontalAlignment = Alignment.CenterHorizontally
        ) {
            if (isLoading) {
                CircularProgressIndicator()
            } else {
                TextField(
                    value = name,
                    onValueChange = { name = it },
                    label = { Text("Nome da Face") },
                    modifier = Modifier.fillMaxWidth()
                )
                Spacer(modifier = Modifier.height(16.dp))

                Button(onClick = {
                    imagePickerLauncher.launch("image/*") // Abre a galeria para selecionar uma imagem
                }) {
                    Text("Selecione uma imagem")
                }

                selectedImageBitmap?.let { bitmap ->
                    Text(text = "Imagem capturada com sucesso!")
                    Spacer(modifier = Modifier.height(8.dp))

                    // Exibe a imagem selecionada
                    Image(
                        bitmap = bitmap.asImageBitmap(),
                        contentDescription = "Imagem capturada",
                        modifier = Modifier
                            .size(200.dp)
                            .padding(16.dp)
                    )

                    // Botão para verificar a face
                    Button(onClick = {
                        when {
                            name.isEmpty() -> {
                                dialogMessage = "Por favor, preencha o nome da face."
                                showDialog = true
                            }
                            selectedImageBitmap == null -> {
                                dialogMessage = "Por favor, selecione uma imagem."
                                showDialog = true
                            }
                            else -> {
                                isLoading = true

                                // Criando o arquivo temporário e convertendo o Bitmap
                                val file = createTempFile(context)
                                val imageFile = convertBitmapToFile(selectedImageBitmap!!, file)

                                // Criando o MultipartBody.Part
                                val requestBody = imageFile.asRequestBody("image/jpeg".toMediaTypeOrNull())
                                val imagePart = MultipartBody.Part.createFormData("verify_image", imageFile.name, requestBody)

                                // Criando o objeto VerifyPersonFace com os dados
                                val verifyPersonFace = VerifyPersonFace(name = name, image = imagePart)

                                // Chama a função da ViewModel para verificar a face
                                apiViewModel.verifyPersonFace(
                                    verifyPersonFace = verifyPersonFace,
                                    onSuccess = { result ->
                                        isLoading = false

                                        val jsonResponse = JSONObject(result)
                                        val verificationError = jsonResponse.getJSONObject("verification_result").getString("verification_error")

                                        if (verificationError.isNotEmpty()) {
                                            dialogMessage = "Erro de verificação: a face informada não existe na base."
                                        } else {
                                            val cosineDist = jsonResponse.getJSONObject("verification_result").getString("cosine_dist")
                                            val similarity = jsonResponse.getJSONObject("verification_result").getString("similarity")
                                            val compareResult = jsonResponse.getJSONObject("verification_result").getString("compare_result")

                                            // Exibe o resultado da verificação
                                            dialogMessage = """
                                                Distância Cosseno: $cosineDist
                                                Similaridade: $similarity
                                                Resultado da Comparação: $compareResult
                                            """.trimIndent()
                                        }
                                        showDialog = true
                                    },
                                    onError = { error ->
                                        isLoading = false
                                        val errorMessage = when {
                                            error.message?.contains("500") == true -> "Erro interno do servidor. Isso pode ocorrer se a imagem não for de uma face válida. Tente novamente com uma nova imagem."
                                            error.message?.contains("400") == true -> "Erro de requisição. Verifique os dados enviados."
                                            else -> {
                                                "Erro ao verificar face: ${error.message}"
                                            }
                                        }
                                        dialogMessage = errorMessage
                                        showDialog = true
                                    }
                                )
                            }
                        }
                    }) {
                        Text("Verificar Face")
                    }

                }
            }

            // Exibe o AlertDialog com as mensagens (de erro ou sucesso)
            if (showDialog) {
                AlertDialog(
                    onDismissRequest = { showDialog = false },
                    title = { Text(text = "Informação") },
                    text = { Text(text = dialogMessage ?: "Sem resultado") },
                    confirmButton = {
                        Button(onClick = { showDialog = false }) {
                            Text("OK")
                        }
                    }
                )
            }
        }
    }
}
```
</details> 

### 4. ApiViewModel

O ApiViewModel centraliza a lógica de comunicação com a API. Ele lida com operações como buscar e cadastrar faces, utilizando o Retrofit.
```kotlin
class ApiViewModel : ViewModel() {
    private val retrofit = Retrofit.Builder()
        .baseUrl("https://137.184.100.1:9450")
        .client(getUnsafeOkHttpClient())
        .addConverterFactory(GsonConverterFactory.create())
        .build()

    private val apiService = retrofit.create(ApiService::class.java)
}

```

Principais Funções:

`fetchPersonFaces:` Busca as faces cadastradas no servidor e atualiza a UI.
      
  ```kotlin
   fun fetchPersonFaces(onSuccess: (List<PersonFace>) -> Unit, onError: (Throwable) -> Unit) {
        viewModelScope.launch {
            try {
                Log.d("ApiViewModel", "Fazendo a requisição para a API...")
                val faces = apiService.getPersonFaces()

                if (faces.isNotEmpty()) {
                    Log.d("ApiViewModel", "Faces recebidas: $faces")
                    onSuccess(faces)
                } else {
                    Log.e("ApiViewModel", "Nenhuma face disponível")
                    onError(Exception("Nenhuma face disponível"))
                }
            } catch (e: Exception) {
                Log.e("ApiViewModel", "Erro na requisição: ${e.message}", e)
                onError(e)
            }
        }
    }
       
  ```
  
`addPersonFace:` Envia uma imagem e nome para cadastrar uma nova face no backend.
  
   ```kotlin
  fun addPersonFace(context: Context, name: String, image: Bitmap, onSuccess: () -> Unit, onError: (Throwable) -> Unit) {
        viewModelScope.launch {
            try {
                val face = Face(name = name, imageUrl = null) // A URL será definida após o upload

                // converte o nome do objeto Face para RequestBody
                val namePart = face.name.toRequestBody("text/plain".toMediaTypeOrNull())

                val file = createTempFile(context)

                // Converte a imagem Bitmap para um arquivo
                val imageFile = convertBitmapToFile(image, file, quality = 50)

                // usa o caminho local temporário da imagem
                face.imageUrl = imageFile.absolutePath

                //multiparbody para fazer uplaod de arquivos
                val imagePart = MultipartBody.Part.createFormData(
                    "person_face", imageFile.name,
                    imageFile.asRequestBody("image/jpeg".toMediaTypeOrNull())
                )

                val response = apiService.addPersonFace(namePart, imagePart)
                if (response.isSuccessful) {
                    face.imageUrl = response.body()?.string()
                    onSuccess()
                } else {
                    onError(Exception("Erro ao cadastrar face: ${response.errorBody()?.string()}"))
                }
            } catch (e: Exception) {
                onError(e)
            }
        }
    }
  ```
`deletePersonFace` Envia um ID para o backend deletar uma face
  
  ```kotlin
  fun deletePersonFace(id: Int, onSuccess: () -> Unit, onError: (Throwable) -> Unit) {
        viewModelScope.launch {
            try {
                val response = apiService.deletePersonFace(id)
                if(response.isSuccessful){
                    onSuccess()
                }else{
                   onError(Exception("Falha ao excluir a face"))
                }
            }
            catch (e: Exception) {
                onError(e)
            }
        }
    }
  ```
 `verifyPersonFace` Envia um nome e uma imagem para o bakcend verificar a face
  
  ```kotlin
  fun verifyPersonFace(
        verifyPersonFace: VerifyPersonFace,
        onSuccess: (String) -> Unit,
        onError: (Throwable) -> Unit
    ) {
        viewModelScope.launch {
            try {
                val namePart = verifyPersonFace.name
                val imagePart = verifyPersonFace.image

                // Faz a requisição para a API
                val response = apiService.verifyPersonFace(namePart, imagePart)

                if (response.isSuccessful) {
                    val responseBody = response.body()?.string() ?: "Resposta vazia"
                    onSuccess(responseBody) // Passa a resposta completa como string para ser exibida
                } else {
                    onError(Exception("${response.errorBody()?.string()}"))
                }
            } catch (e: Exception) {
                onError(e)
            }
        }
    }
  ```


## 5. Funções de utilidade

### 5.1 getUnsafeOkHttpClient
Essa função cria um cliente `OkHttp` que desativa a verificação de certificados SSL. Isso pode ser útil para testes em ambientes de desenvolvimento com certificados auto-assinados, **mas não deve ser usado em produção**.

```kotlin
fun getUnsafeOkHttpClient(): OkHttpClient {
    val trustAllCerts = arrayOf<TrustManager>(object : X509TrustManager {
        override fun checkClientTrusted(chain: Array<X509Certificate>, authType: String) {}
        override fun checkServerTrusted(chain: Array<X509Certificate>, authType: String) {}
        override fun getAcceptedIssuers(): Array<X509Certificate> = arrayOf()
    })
    
    val sslContext = SSLContext.getInstance("SSL").apply {
        init(null, trustAllCerts, java.security.SecureRandom())
    }

    return OkHttpClient.Builder()
        .sslSocketFactory(sslContext.socketFactory, trustAllCerts[0] as X509TrustManager)
        .hostnameVerifier { _, _ -> true }
        .build()
}

```
### 5.2 convertBitmapToFile
Essa função converte um objeto `Bitmap` em um arquivo físico no sistema, o que é necessário para enviar a imagem da face via multipart para a API.

```kotlin
fun convertBitmapToFile(bitmap: Bitmap, file: File, quality: Int = 50): File {
    try {
        val fileOutputStream = FileOutputStream(file)
        bitmap.compress(Bitmap.CompressFormat.JPEG, quality, fileOutputStream)
        fileOutputStream.flush()
        fileOutputStream.close()
    } catch (e: IOException) {
        e.printStackTrace()
    }
    return file
}

```

### 5.3 createTempFile
Essa função cria um arquivo temporário no cache da aplicação para armazenar a imagem capturada da câmera antes de ser enviada à API.

```kotlin
fun createTempFile(context: Context): File {
    return File(context.cacheDir, "temp_image.jpg").apply {
        createNewFile()
    }
}


```

### 5.4 imageProxyToBitmap
Essa função converte o objeto `ImageProxy` (retornado pela câmera) em um objeto `Bitmap`, que pode ser exibido na interface do usuário e enviado ao backend.

```kotlin
private fun imageProxyToBitmap(image: ImageProxy): Bitmap {
    val buffer: ByteBuffer = image.planes[0].buffer
    val bytes = ByteArray(buffer.remaining())
    buffer.get(bytes)
    return BitmapFactory.decodeByteArray(bytes, 0, bytes.size)
}
```

### 5.5 CameraPreviewView
A função `CameraPreviewView` é uma composable do Jetpack Compose que exibe a visualização da câmera frontal e permite capturar uma imagem. Ela utiliza o `PreviewView` para exibir a pré-visualização da câmera e o `ImageCapture` para tirar a foto. O ciclo de vida da câmera é vinculado ao ciclo de vida da interface utilizando o `ProcessCameraProvider`. Quando o usuário clica no botão "Capturar Face", a imagem capturada é convertida em um `Bitmap` e passada para o `callback` fornecido, permitindo o processamento posterior da imagem.

```kotlin
@Composable
fun CameraPreviewView(modifier: Modifier = Modifier, onImageCaptured: (Bitmap) -> Unit) {
    val context = LocalContext.current
    val lifecycleOwner: LifecycleOwner = LocalLifecycleOwner.current

    val previewView = PreviewView(context)
    val imageCapture = ImageCapture.Builder().build()
    val cameraProviderFuture = ProcessCameraProvider.getInstance(context)

    LaunchedEffect(Unit) {
        val cameraProvider = cameraProviderFuture.get()
        val preview = androidx.camera.core.Preview.Builder().build().also {
            it.setSurfaceProvider(previewView.surfaceProvider)
        }

        try {
            cameraProvider.unbindAll()
            cameraProvider.bindToLifecycle(
                lifecycleOwner,
                CameraSelector.DEFAULT_FRONT_CAMERA,
                preview,
                imageCapture
            )
        } catch (e: Exception) {
            e.printStackTrace()
        }
    }

    AndroidView(
        factory = { previewView },
        modifier = modifier
    )

    Button(
        onClick = {
            imageCapture.takePicture(
                ContextCompat.getMainExecutor(context),
                object : ImageCapture.OnImageCapturedCallback() {
                    override fun onCaptureSuccess(image: ImageProxy) {
                        val bitmap = imageProxyToBitmap(image)
                        onImageCaptured(bitmap)
                        image.close()
                    }

                    override fun onError(exception: ImageCaptureException) {
                        println("Erro ao capturar imagem: ${exception.message}")
                    }
                }
            )
        },
        modifier = Modifier.padding(top = 16.dp)
    ) {
        Text("Capturar Face")
    }
}
```

### 5.6 getBitmapFromUri
A função `getBitmapFromUri` converte um `Uri` em um objeto `Bitmap` a partir de um recurso de imagem, utilizando o `ContentResolver` para acessar o arquivo. Ela recebe um `Uri`, o contexto da aplicação, e os valores de largura e altura máximos. Após decodificar a imagem original, a função redimensiona o `Bitmap` para se ajustar aos limites fornecidos, utilizando a função auxiliar `resizeBitmap`. Caso o Bitmap original seja nulo, a função retorna null.

```kotlin
fun getBitmapFromUri(uri: Uri, context: Context, maxWidth: Int, maxHeight: Int): Bitmap? {
    val inputStream = context.contentResolver.openInputStream(uri)
    val originalBitmap = BitmapFactory.decodeStream(inputStream)
    inputStream?.close()

    return originalBitmap?.let {
        resizeBitmap(it, maxWidth, maxHeight)
    }
}
```

### 5.7 resizeBitmap
A função `resizeBitmap` redimensiona um `Bitmap` original mantendo sua proporção de aspecto. Ela recebe o `Bitmap` original, uma largura e uma altura máximas. A função calcula a nova largura e altura com base no aspecto da imagem, ajustando a maior dimensão ao limite máximo permitido e escalando a outra dimensão proporcionalmente. O Bitmap redimensionado é retornado usando a função Bitmap.createScaledBitmap, preservando a qualidade da imagem.

```kotlin
fun resizeBitmap(original: Bitmap, maxWidth: Int, maxHeight: Int): Bitmap {
    val width = original.width
    val height = original.height

    val aspectRatio: Float = width.toFloat() / height.toFloat()
    val newWidth: Int
    val newHeight: Int

    if (width > height) {
        newWidth = maxWidth
        newHeight = (newWidth / aspectRatio).toInt()
    } else {
        newHeight = maxHeight
        newWidth = (newHeight * aspectRatio).toInt()
    }

    return Bitmap.createScaledBitmap(original, newWidth, newHeight, true)
}
```

## 6. Components 

A camada de **components** contém componentes reutilizáveis que são usados nas telas do aplicativo para criar uma interface de usuário consistente. Esses componentes são projetados para serem independentes e reutilizáveis em diferentes partes da aplicação.

### 6.1 DropdownMenu

O arquivo `DropdownMenu.kt` define o menu suspenso que é acionado na barra de navegação. Ele fornece opções de navegação adicionais, como "Faces" e "Cadastrar Face". Esse componente é exibido quando o botão de menu na IconButton, que se encontra dentro do menu `Header` é pressionado.
```kotlin
@Composable
fun DropdownMenu(
    navController: NavController,
    expanded: Boolean,
    onDismissRequest: () -> Unit,
) {
    androidx.compose.material3.DropdownMenu(
        expanded = expanded,
        onDismissRequest = onDismissRequest,
        content = {
            DropdownMenuItem(
                text = { Text(text = "Faces") },
                leadingIcon = {
                    Icon(
                        imageVector = Icons.Outlined.Person,
                        contentDescription = null,
                        tint = MaterialTheme.colorScheme.primary,
                    )
                },
                onClick = {
                    navController.navigate(AppScreen.HomeScreen.route)
                    onDismissRequest()
                }
            )
            DropdownMenuItem(
                text = { Text(text = "Cadastrar Face") },
                leadingIcon = {
                    Icon(
                        imageVector = Icons.Outlined.Face,
                        contentDescription = null,
                        tint = MaterialTheme.colorScheme.primary,
                    )
                },
                onClick = {
                    navController.navigate(AppScreen.AddFaceScreen.route)
                    onDismissRequest()
                }
            )
            DropdownMenuItem(
                text = { Text(text = "Verificar Face") },
                leadingIcon = {
                    Icon(
                        imageVector = Icons.Outlined.AccountBox,
                        contentDescription = null,
                        tint = MaterialTheme.colorScheme.primary,
                    )
                },
                onClick = {
                    navController.navigate(AppScreen.VerifyScreen.route)
                    onDismissRequest()
                }
            )
        }
    )
}
```
## 6.2 Header

O arquivo `Header.kt` define um componente de cabeçalho reutilizável que inclui um título e um botão de menu. Foi separado como um componente reutilizável específico para ser usado em telas que requerem uma navegação mais simples.
```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun Header(navController: NavController, title: String) {
    var expanded by remember { mutableStateOf(false) }

    TopAppBar(
        title = {
            Text(
                text = title,
                color = MaterialTheme.colorScheme.onPrimary
            )
        },
        actions = {
            Box {
                IconButton(onClick = {
                    expanded = !expanded
                }) {
                    Icon(
                        Icons.Outlined.Menu,
                        contentDescription = "Menu",
                        tint = Color.White
                    )
                }
                DropdownMenu(
                    expanded = expanded,
                    onDismissRequest = { expanded = false },
                    navController = navController
                )
            }
        },
        colors = TopAppBarDefaults.topAppBarColors(
            containerColor = MaterialTheme.colorScheme.primary
        )
    )
}

```


### Considerações finais

A arquitetura MVVM utilizada separa bem a lógica de negócio da interface de usuário, facilitando a manutenção e evolução do código. O uso de componentes reutilizáveis, como `DropdownMenu` e `Header`, também ajuda a manter a consistência visual e de navegação.
