# Lab 13 — Application Android de Géolocalisation avec OpenStreetMap

## Objectif du Lab

Ce laboratoire a pour objectif de réaliser une application Android en Java permettant de :

* récupérer la position géographique d’un appareil Android ;
* envoyer les coordonnées vers un serveur PHP/MySQL ;
* enregistrer les positions dans une base de données ;
* afficher les positions enregistrées sur une carte OpenStreetMap avec des marqueurs ;
* utiliser Volley pour la communication entre Android et le serveur.

L’application est composée de deux parties :

1. **Partie mobile Android** : récupération GPS, envoi HTTP et affichage de la carte.
2. **Partie backend PHP/MySQL** : réception, stockage et récupération des positions.

---

## Prérequis

Avant de commencer, il faut avoir :

* Android Studio installé ;
* des connaissances de base en Android avec Java ;
* un émulateur Android ou un appareil physique ;
* un serveur local comme XAMPP ou WAMP ;
* une base de données MySQL ;
* une connexion Internet pour charger les tuiles OpenStreetMap.

---

# Étape 1 — Configuration du Projet Android

## 1.1 Création du projet

Dans Android Studio :

1. Cliquer sur **New Project**.
2. Choisir **Empty Activity**.
3. Configurer le projet :

   * Nom : `MapApplication`
   * Package name : `com.example.mapapplication`
   * Langage : `Java`
   * Minimum SDK : `API 24 Android 7.0`
4. Cliquer sur **Finish**.

---

## 1.2 Configuration des dépendances Gradle

Ouvrir le fichier `app/build.gradle.kts` et ajouter les dépendances nécessaires.

```kotlin
plugins {
    alias(libs.plugins.android.application)
}

android {
    namespace = "com.example.mapapplication"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.example.mapapplication"
        minSdk = 24
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            isMinifyEnabled = false
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_1_8
        targetCompatibility = JavaVersion.VERSION_1_8
    }
}

dependencies {
    implementation(libs.appcompat)
    implementation(libs.material)
    implementation(libs.activity)
    implementation(libs.constraintlayout)

    testImplementation(libs.junit)
    androidTestImplementation(libs.ext.junit)
    androidTestImplementation(libs.espresso.core)

    implementation(libs.volley)
    implementation(libs.maps.core)
    implementation(libs.osmdroid)
}
```

### Explication

Les dépendances principales utilisées sont :

* `Volley` : pour envoyer et recevoir des requêtes HTTP.
* `OSMDroid` : pour afficher une carte OpenStreetMap dans Android.
* `AppCompat`, `Material`, `ConstraintLayout` : pour créer une interface Android moderne.

---

## 1.3 Configuration du fichier Manifest

Ouvrir le fichier `AndroidManifest.xml` et ajouter les permissions nécessaires.

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.READ_PHONE_STATE" />
    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:allowBackup="true"
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="@xml/backup_rules"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:networkSecurityConfig="@xml/network_security_config"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.MapApplication"
        tools:targetApi="31">

        <activity
            android:name=".GoogleMapActivity"
            android:exported="false" />

        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

    </application>

</manifest>
```

### Explication des permissions

* `ACCESS_FINE_LOCATION` : permet d’obtenir une localisation précise avec le GPS.
* `ACCESS_COARSE_LOCATION` : permet d’obtenir une localisation approximative.
* `READ_PHONE_STATE` : utilisée traditionnellement pour récupérer l’IMEI.
* `INTERNET` : permet à l’application de communiquer avec le serveur et de charger la carte.

Depuis Android 6.0, les permissions de localisation doivent être demandées aussi pendant l’exécution de l’application.

---

## 1.4 Configuration de sécurité réseau

Créer le fichier suivant :

`app/src/main/res/xml/network_security_config.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="true">10.0.2.2</domain>
    </domain-config>
</network-security-config>
```

### Explication

L’adresse `10.0.2.2` permet à l’émulateur Android d’accéder au serveur local de la machine hôte.

Cette configuration autorise le trafic HTTP pendant le développement.

En production, il est recommandé d’utiliser HTTPS.

---

# Étape 2 — Création de l’Interface Utilisateur

## 2.1 Layout de l’activité principale

Modifier le fichier `activity_main.xml`.

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <Button
        android:id="@+id/btnMap"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Afficher La Map"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.5"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintVertical_bias="0.5" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

### Explication

Cette interface contient un seul bouton centré. Ce bouton permet d’ouvrir l’activité contenant la carte.

---

## 2.2 Layout de l’activité carte

Créer le fichier `activity_google_map.xml`.

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".GoogleMapActivity">

    <org.osmdroid.views.MapView
        android:id="@+id/map"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

### Explication

Le composant `MapView` d’OSMDroid occupe tout l’écran pour afficher la carte OpenStreetMap.

---

## 2.3 Ajout du marqueur

Ajouter une image de marqueur dans le dossier :

```text
app/src/main/res/drawable/marker.png
```

Cette image sera utilisée pour représenter les positions sur la carte.

---

# Étape 3 — Implémentation de MainActivity

Modifier le fichier `MainActivity.java`.

```java
package com.example.mapapplication;

import android.Manifest;
import android.content.Context;
import android.content.Intent;
import android.content.pm.PackageManager;
import android.location.Location;
import android.location.LocationListener;
import android.location.LocationManager;
import android.os.Bundle;
import android.provider.Settings;
import android.view.View;
import android.widget.Button;
import android.widget.Toast;

import androidx.annotation.NonNull;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.app.ActivityCompat;
import androidx.core.content.ContextCompat;

import com.android.volley.AuthFailureError;
import com.android.volley.Request;
import com.android.volley.RequestQueue;
import com.android.volley.Response;
import com.android.volley.VolleyError;
import com.android.volley.toolbox.StringRequest;
import com.android.volley.toolbox.Volley;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;

public class MainActivity extends AppCompatActivity {

    private Button btnMap;
    private double latitude;
    private double longitude;
    private double altitude;
    private float accuracy;

    RequestQueue requestQueue;
    String insertUrl = "http://10.0.2.2/map_project/createPosition.php";
    LocationManager locationManager;

    private static final int PERMISSION_REQUEST_CODE = 100;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        requestQueue = Volley.newRequestQueue(getApplicationContext());
        locationManager = (LocationManager) getSystemService(Context.LOCATION_SERVICE);

        btnMap = findViewById(R.id.btnMap);

        btnMap.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                startActivity(new Intent(MainActivity.this, GoogleMapActivity.class));
            }
        });

        if (ContextCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION) != PackageManager.PERMISSION_GRANTED
                || ContextCompat.checkSelfPermission(this, Manifest.permission.READ_PHONE_STATE) != PackageManager.PERMISSION_GRANTED) {

            ActivityCompat.requestPermissions(this,
                    new String[]{
                            Manifest.permission.ACCESS_FINE_LOCATION,
                            Manifest.permission.ACCESS_COARSE_LOCATION,
                            Manifest.permission.READ_PHONE_STATE
                    }, PERMISSION_REQUEST_CODE);
        } else {
            startLocationUpdates();
        }
    }

    private void startLocationUpdates() {
        if (ActivityCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION) != PackageManager.PERMISSION_GRANTED
                && ActivityCompat.checkSelfPermission(this, Manifest.permission.ACCESS_COARSE_LOCATION) != PackageManager.PERMISSION_GRANTED) {
            return;
        }

        locationManager.requestLocationUpdates(LocationManager.GPS_PROVIDER, 60000, 150, new LocationListener() {
            @Override
            public void onLocationChanged(@NonNull Location location) {
                latitude = location.getLatitude();
                longitude = location.getLongitude();
                altitude = location.getAltitude();
                accuracy = location.getAccuracy();

                String msg = String.format(
                        getResources().getString(R.string.new_location),
                        latitude,
                        longitude,
                        altitude,
                        accuracy
                );

                addPosition(latitude, longitude);
                Toast.makeText(getApplicationContext(), msg, Toast.LENGTH_LONG).show();
            }

            @Override
            public void onStatusChanged(String provider, int status, Bundle extras) {
            }

            @Override
            public void onProviderEnabled(@NonNull String provider) {
            }

            @Override
            public void onProviderDisabled(@NonNull String provider) {
            }
        });
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);

        if (requestCode == PERMISSION_REQUEST_CODE) {
            if (grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                startLocationUpdates();
            } else {
                Toast.makeText(this, "Permission refusée. L'application ne peut pas fonctionner.", Toast.LENGTH_SHORT).show();
            }
        }
    }

    void addPosition(final double lat, final double lon) {
        StringRequest request = new StringRequest(Request.Method.POST,
                insertUrl,
                new Response.Listener<String>() {
                    @Override
                    public void onResponse(String response) {
                    }
                },
                new Response.ErrorListener() {
                    @Override
                    public void onErrorResponse(VolleyError error) {
                    }
                }) {
            @Override
            protected Map<String, String> getParams() throws AuthFailureError {
                HashMap<String, String> params = new HashMap<>();
                SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

                params.put("latitude", lat + "");
                params.put("longitude", lon + "");
                params.put("date", sdf.format(new Date()));

                String androidId = Settings.Secure.getString(
                        getContentResolver(),
                        Settings.Secure.ANDROID_ID
                );

                params.put("imei", androidId);

                return params;
            }
        };

        requestQueue.add(request);
    }
}
```

### Explication

Cette classe permet de :

* demander les permissions de localisation ;
* écouter les changements de position avec `LocationManager` ;
* récupérer la latitude, la longitude, l’altitude et la précision ;
* envoyer la position vers le serveur avec Volley ;
* ouvrir l’activité de la carte avec le bouton `Afficher La Map`.

L’identifiant utilisé est `ANDROID_ID`, qui remplace l’IMEI pour éviter les restrictions modernes d’Android.

---

# Étape 4 — Implémentation de GoogleMapActivity

Créer le fichier `GoogleMapActivity.java`.

```java
package com.example.mapapplication;

import android.graphics.Bitmap;
import android.graphics.drawable.BitmapDrawable;
import android.graphics.drawable.Drawable;
import android.os.Bundle;
import android.widget.Toast;

import androidx.appcompat.app.AppCompatActivity;

import com.android.volley.Request;
import com.android.volley.RequestQueue;
import com.android.volley.Response;
import com.android.volley.VolleyError;
import com.android.volley.toolbox.JsonObjectRequest;
import com.android.volley.toolbox.Volley;

import org.json.JSONArray;
import org.json.JSONException;
import org.json.JSONObject;

import org.osmdroid.config.Configuration;
import org.osmdroid.tileprovider.tilesource.TileSourceFactory;
import org.osmdroid.util.GeoPoint;
import org.osmdroid.views.MapView;
import org.osmdroid.views.overlay.Marker;

public class GoogleMapActivity extends AppCompatActivity {

    private MapView map;
    private RequestQueue requestQueue;
    private String showUrl = "http://10.0.2.2/map_project/getPosition.php";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        Configuration.getInstance().load(
                getApplicationContext(),
                getSharedPreferences("prefs", MODE_PRIVATE)
        );

        setContentView(R.layout.activity_google_map);

        map = findViewById(R.id.map);
        map.setTileSource(TileSourceFactory.MAPNIK);
        map.setBuiltInZoomControls(true);
        map.setMultiTouchControls(true);

        map.getController().setZoom(15.0);
        map.getController().setCenter(new GeoPoint(37.272525, -122.12106));

        requestQueue = Volley.newRequestQueue(getApplicationContext());

        loadPositions();
    }

    private void loadPositions() {
        JsonObjectRequest jsonObjectRequest = new JsonObjectRequest(
                Request.Method.POST,
                showUrl,
                null,
                new Response.Listener<JSONObject>() {
                    @Override
                    public void onResponse(JSONObject response) {
                        try {
                            JSONArray positions = response.getJSONArray("positions");

                            for (int i = 0; i < positions.length(); i++) {
                                JSONObject position = positions.getJSONObject(i);
                                double lat = position.getDouble("latitude");
                                double lng = position.getDouble("longitude");

                                Marker marker = new Marker(map);
                                marker.setPosition(new GeoPoint(lat, lng));
                                marker.setTitle("Marker " + (i + 1));

                                Drawable original = getResources().getDrawable(R.drawable.marker);
                                Bitmap bitmap = ((BitmapDrawable) original).getBitmap();
                                Bitmap scaledBitmap = Bitmap.createScaledBitmap(bitmap, 80, 80, false);

                                marker.setIcon(new BitmapDrawable(getResources(), scaledBitmap));
                                marker.setAnchor(Marker.ANCHOR_CENTER, Marker.ANCHOR_BOTTOM);

                                map.getOverlays().add(marker);

                                Toast.makeText(
                                        getApplicationContext(),
                                        "Lat : " + lat + " lng : " + lng,
                                        Toast.LENGTH_SHORT
                                ).show();
                            }

                            map.invalidate();

                        } catch (JSONException e) {
                            e.printStackTrace();
                        }
                    }
                },
                new Response.ErrorListener() {
                    @Override
                    public void onErrorResponse(VolleyError error) {
                        error.printStackTrace();
                    }
                }
        );

        requestQueue.add(jsonObjectRequest);
    }
}
```

### Explication

Cette activité permet de :

* charger une carte OpenStreetMap avec OSMDroid ;
* récupérer les positions depuis le serveur PHP ;
* lire les données JSON reçues ;
* créer un marqueur pour chaque position ;
* afficher les marqueurs sur la carte.

Le fichier utilise `JsonObjectRequest` pour récupérer les positions au format JSON.

---

# Étape 5 — Création de la Partie Backend PHP/MySQL

## 5.1 Création de la base de données

Dans phpMyAdmin, créer une base de données nommée :

```sql
CREATE DATABASE map_project;
```

Ensuite, créer la table `positions`.

```sql
CREATE TABLE `positions` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `latitude` double NOT NULL,
  `longitude` double NOT NULL,
  `date` datetime NOT NULL,
  `imei` varchar(50) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### Explication

La table `positions` contient :

* `id` : identifiant unique de chaque position ;
* `latitude` : latitude de l’appareil ;
* `longitude` : longitude de l’appareil ;
* `date` : date et heure de l’enregistrement ;
* `imei` : identifiant de l’appareil, ici remplacé par `ANDROID_ID`.

---

## 5.2 Script PHP pour enregistrer les positions

Créer le fichier suivant :

```text
htdocs/map_project/createPosition.php
```

```php
<?php
$host = "localhost";
$db_name = "map_project";
$username = "root";
$password = "";

try {
    $conn = new PDO("mysql:host=$host;dbname=$db_name", $username, $password);
    $conn->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
} catch(PDOException $e) {
    echo "Erreur de connexion: " . $e->getMessage();
    die();
}

$latitude = isset($_POST['latitude']) ? $_POST['latitude'] : null;
$longitude = isset($_POST['longitude']) ? $_POST['longitude'] : null;
$date = isset($_POST['date']) ? $_POST['date'] : null;
$imei = isset($_POST['imei']) ? $_POST['imei'] : null;

if($latitude === null || $longitude === null || $date === null || $imei === null) {
    echo json_encode(["success" => false, "message" => "Données manquantes"]);
    exit;
}

try {
    $stmt = $conn->prepare("INSERT INTO positions (latitude, longitude, date, imei) VALUES (:latitude, :longitude, :date, :imei)");

    $stmt->bindParam(':latitude', $latitude);
    $stmt->bindParam(':longitude', $longitude);
    $stmt->bindParam(':date', $date);
    $stmt->bindParam(':imei', $imei);

    $stmt->execute();

    echo json_encode(["success" => true, "message" => "Position enregistrée avec succès"]);
} catch(PDOException $e) {
    echo json_encode(["success" => false, "message" => "Erreur: " . $e->getMessage()]);
}
?>
```

### Explication

Ce script reçoit les données envoyées par Android avec la méthode POST.

Il récupère :

* la latitude ;
* la longitude ;
* la date ;
* l’identifiant de l’appareil.

Ensuite, il insère ces données dans la table `positions`.

L’utilisation de `PDO` et des requêtes préparées permet d’éviter les injections SQL.

---

## 5.3 Script PHP pour récupérer les positions

Créer le fichier suivant :

```text
htdocs/map_project/getPosition.php
```

```php
<?php
$host = "localhost";
$db_name = "map_project";
$username = "root";
$password = "";

try {
    $conn = new PDO("mysql:host=$host;dbname=$db_name", $username, $password);
    $conn->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
} catch(PDOException $e) {
    echo json_encode(["success" => false, "message" => "Erreur de connexion: " . $e->getMessage()]);
    die();
}

try {
    $stmt = $conn->prepare("SELECT * FROM positions ORDER BY date DESC");
    $stmt->execute();

    $positions = $stmt->fetchAll(PDO::FETCH_ASSOC);

    echo json_encode(["success" => true, "positions" => $positions]);
} catch(PDOException $e) {
    echo json_encode(["success" => false, "message" => "Erreur: " . $e->getMessage()]);
}
?>
```

### Explication

Ce script récupère toutes les positions enregistrées dans la base de données.

Les données sont retournées sous forme JSON pour être exploitées par l’application Android.

La réponse contient un tableau nommé `positions`.

---

# Étape 6 — Test de l’Application

## 6.1 Lancer l’application

Pour tester l’application :

1. Démarrer XAMPP ou WAMP.
2. Lancer Apache et MySQL.
3. Vérifier que la base `map_project` existe.
4. Vérifier que les fichiers PHP sont dans le bon dossier.
5. Lancer l’application depuis Android Studio.
6. Accepter les permissions de localisation.

---

## 6.2 Vérifier l’enregistrement des positions

Quand la position change, l’application envoie automatiquement les coordonnées au serveur.

Pour vérifier :

1. Ouvrir phpMyAdmin.
2. Aller dans la base `map_project`.
3. Ouvrir la table `positions`.
4. Vérifier que les lignes sont bien ajoutées.

---

## 6.3 Vérifier l’affichage de la carte

Cliquer sur le bouton :

```text
Afficher La Map
```

L’application ouvre l’activité carte et affiche les positions enregistrées sous forme de marqueurs.

---

# Bonnes Pratiques

## Optimisation de la localisation

Il est possible d’améliorer la gestion GPS en :

* réduisant la fréquence des mises à jour ;
* utilisant `NETWORK_PROVIDER` pour économiser la batterie ;
* arrêtant les mises à jour lorsque l’application est en arrière-plan.

---

## Optimisation de la carte

Pour améliorer l’affichage de la carte :

* limiter le nombre de marqueurs affichés ;
* utiliser un système de regroupement de marqueurs ;
* gérer correctement le cache des tuiles ;
* supprimer les Toasts de débogage en production.

---

## Sécurité

Pour sécuriser l’application :

* utiliser HTTPS au lieu de HTTP ;
* valider les données côté serveur ;
* éviter de stocker des données sensibles ;
* ajouter une authentification si nécessaire ;
* protéger les API PHP.

---

# Résultat Final

À la fin de ce Lab, l’application permet de :

* récupérer la position GPS d’un appareil Android ;
* envoyer la latitude et la longitude à un serveur PHP ;
* enregistrer les données dans MySQL ;
* récupérer les positions depuis la base de données ;
* afficher les positions sur une carte OpenStreetMap avec des marqueurs.

---

# Conclusion

Ce Lab permet de construire une application Android complète basée sur la géolocalisation.

Il montre comment connecter une application mobile à un backend PHP/MySQL, comment gérer les permissions Android, comment envoyer des données avec Volley et comment afficher des positions sur une carte OpenStreetMap.

Les notions principales vues dans ce Lab sont :

* permissions Android ;
* localisation GPS ;
* Volley ;
* PHP/MySQL ;
* JSON ;
* OSMDroid ;
* affichage de marqueurs sur une carte.

Cette application peut servir de base pour des projets plus avancés comme le suivi en temps réel, la gestion de flotte, la livraison ou les applications de navigation.
