---
author: Saurabh Dalakoti
pubDatetime: 2024-01-16T20:40:08Z
modDatetime: 2024-07-22T16:14:28Z
title: Using biometric login in android
featured: false
draft: false
tags:
  - Android
  - Mobile
description: How to use biometric login apis in android compose
---

## Why Biometric login

Biometric credentials are not shared to server, unlike JWt auth token, login and password. It is just enabled on client side, and if client (in our case android OS) says its ok, we would get into app.

Some other benefits includes

- enhanced security
  - uniqueness, as biometric like face, fingerprint is unique to every individual
  - hard to spoof biometric results
- improved convenience
  - better speed as compared to typed password
  - simply a touch, or a look

## Adding dependency

```kts
implementation("androidx.biometric:biometric:1.2.0-alpha05")
implementation("androidx.appcompat:appcompat:1.7.0-alpha03")
```

## Code changes

- make your compose activity from `ComponentActivity` to `AppCompatActivity`
- earlier it was showing content view as usual
  - but now it would be lock screen or content screen depending upon lock screen state
  - now this locked or unlocked screen would be stored in VM or repo
  - since I am working on a single activity architecture it would be easy peassy

```kotlin
setContent {
			MainScreen(
				onShowOSBiometricsModal = {
					authenticateWithOSBiometricsModal(
						biometricPromptCallback = handleBiometricAuthResult(),
					)
				},
				onContinueWithoutAuthentication = {
					// todo remove
				},
				userManager
			)
		}

```

Some more code to handle

- inactivity code when `mainActivity` is paused, so that screen can be locked
- handling BioMetric result callback and asking repository to unlock the screen
  - and depending upon this compose would re-render and would should content screen instead of lock screen

```kotlin

	override fun onPause() {
		Log.d(TAG, "onPause: called")
		super.onPause()
		userManager.startUserInactiveTimeCounter()
	}

	private fun handleBiometricAuthResult(
		onAuthSuccess: () -> Unit = {}
	): BiometricPrompt.AuthenticationCallback {
		return object : BiometricPrompt.AuthenticationCallback() {
			override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
				userManager.markScreenAsUnlocked()
				onAuthSuccess()
			}

			override fun onAuthenticationFailed() {}
			override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {}
		}
	}

	private fun authenticateWithOSBiometricsModal(
		biometricPromptCallback: BiometricPrompt.AuthenticationCallback
	) {
		val executor = ContextCompat.getMainExecutor(this)
		val biometricPrompt = BiometricPrompt(
			this,
			executor,
			biometricPromptCallback
		)

		val promptInfo = BiometricPrompt.PromptInfo.Builder()
			.setTitle(
				getString(R.string.authentication_required)
			)
			.setSubtitle(
				getString(R.string.authentication_required_description)
			)
			.setAllowedAuthenticators(
				BiometricManager.Authenticators.BIOMETRIC_WEAK or
					BiometricManager.Authenticators.DEVICE_CREDENTIAL
			)
			.setConfirmationRequired(false)
			.build()

		biometricPrompt.authenticate(promptInfo)
	}

```

Some more code which does following

- shows biometric auth prompt when not signed in
- when signed in, show navGraph as per logged In state

```kotlin

@Composable
fun MainScreen(
	onShowOSBiometricsModal: () -> Unit,
	onContinueWithoutAuthentication: () -> Unit,
	userManager: UserManager
) {
	LaunchedEffect(key1 = Unit, block = {
		FirebaseMessagingTokenLogger().apply {
			logToken()
		}
	})
	val currentUser by remember {
		mutableStateOf(userManager.getCurrentUser())
	}
	val isLoggedIn by remember(key1 = currentUser) {
		mutableStateOf(currentUser != null)
	}
	val isAppLocked by userManager.isAppLocked.collectAsStateWithLifecycle()
	when (isAppLocked) {
		true -> {
			UniverseTheme {
				FingerPrint(
					onShowOSBiometricsModal = onShowOSBiometricsModal,
					onContinueWithoutAuthentication = onContinueWithoutAuthentication,
				)
			}
		}

		false -> {
			UniverseTheme {
				DestinationsNavHost(
					navGraph = when (isLoggedIn) {
						true -> {
							NavGraphs.homeGraph
						}

						false -> {
							NavGraphs.root
						}
	@@ -44,3 +165,117 @@ class MainActivity : ComponentActivity() {
		}
	}
}

@Composable
fun FingerPrint(
	onShowOSBiometricsModal: () -> Unit,
	onContinueWithoutAuthentication: () -> Unit,
) {
	val latestOnContinueWithoutAuthentication by rememberUpdatedState(
		newValue = onContinueWithoutAuthentication
	)
	val latestOnShowOSBiometricsModal by rememberUpdatedState(onShowOSBiometricsModal)

	val context = LocalContext.current
	LaunchedEffect(key1 = Unit, block = {
		osAuthentication(
			context = context,
			onShowOSBiometricsModal = latestOnShowOSBiometricsModal,
			onContinueWithoutAuthentication = latestOnContinueWithoutAuthentication
		)
	})
	Column(
		modifier = Modifier
			.fillMaxSize()
			.background(
				color = UniverseTheme.colors.primary,
			),
		horizontalAlignment = Alignment.CenterHorizontally,
	) {
		Text(
			text = buildAnnotatedString {
				withStyle(
					style = SpanStyle(
						fontSize = 80.sp,
						color = UniverseTheme.colors.onSecondary,
					),
				) {
					append("NFSW")
				}
				withStyle(
					style = SpanStyle(
						fontSize = 20.sp,
						color = UniverseTheme.colors.onSecondary,
					),
				) {
					append("content")
				}
			},
			modifier = Modifier
				.padding(
					vertical = 10.dp,
				),
			textAlign = TextAlign.Center,
			style = UniverseTheme.typography.headlineLarge,
		)
		Text(
			text = "App is locked",
			modifier = Modifier
				.fillMaxWidth()
				.padding(
					vertical = 10.dp,
				),
			color = UniverseTheme.colors.onSecondary,
			textAlign = TextAlign.Center,
			style = UniverseTheme.typography.headlineMedium,
		)

		Image(
			modifier = Modifier
				.clickable {
					osAuthentication(
						context = context,
						onShowOSBiometricsModal = latestOnShowOSBiometricsModal,
						onContinueWithoutAuthentication = latestOnContinueWithoutAuthentication
					)
				}
				.size(width = 96.dp, height = 138.dp),
			painter = painterResource(id = R.drawable.ic_fingerprint),
			contentScale = ContentScale.FillBounds,
			contentDescription = "unlock icon",
			colorFilter = ColorFilter.tint(
				color = UniverseTheme.colors.onPrimary,
			),
		)
	}
}

private fun osAuthentication(
	context: Context,
	onShowOSBiometricsModal: () -> Unit,
	onContinueWithoutAuthentication: () -> Unit
) {
	if (hasLockScreen(context)) {
		onShowOSBiometricsModal()
	} else {
		onContinueWithoutAuthentication()
	}
}

@Preview("default")
@Preview("dark theme", uiMode = Configuration.UI_MODE_NIGHT_YES)
@Preview("large font", fontScale = 2f)
@Composable
fun PreviewFingerPrint() {
	UniverseTheme {
		FingerPrint(
			onContinueWithoutAuthentication = {},
			onShowOSBiometricsModal = {},
		)
	}
}

fun hasLockScreen(context: Context): Boolean {
	val keyguardManager = context.getSystemService(Context.KEYGUARD_SERVICE) as KeyguardManager
	return keyguardManager.isDeviceSecure
}

```

Since you have migrated from ComposeActivity to Androidx activity theming also need to be changed
from
`<style name="AppTheme" parent="android:Theme.Material.NoActionBar"/>`
to
`<style name="AppTheme" parent="Theme.AppCompat.DayNight.NoActionBar"/>`

In your user-manager repository u can have something like this

```kotlin

class UserManager(
	private val realmRepository: RealmRepository,
) {

	// Time in seconds
	private val userTimeInActivityTime = 20
	private val _isAppLocked = MutableStateFlow(true)
	val isAppLocked: StateFlow<Boolean>
		get() = _isAppLocked

	private val scope = CoroutineScope(Dispatchers.Default)
	private var userInactiveTime = 0
	private var userInactiveJob: Job? = null

	fun startUserInactiveTimeCounter() {
		if (userInactiveJob != null && userInactiveJob!!.isActive) return

		userInactiveJob = scope.launch {
			while (userInactiveTime < userTimeInActivityTime &&
				userInactiveJob != null && !userInactiveJob?.isCancelled!!
			) {
				delay(1000)
				userInactiveTime += 1
			}

			if (!isAppLocked.value) {
				_isAppLocked.value = true
			}
			cancel()
		}
	}

	fun markScreenAsUnlocked() {
		_isAppLocked.value = false
	}
}

```

In multi activity architecture, where u have multiple activity, instead of starting `startUserInactiveTimeCounter` on onStop of `mainActivity`, you can listen to application process lifecycle change in app class, and start reacting on it. That would work like a charm.
