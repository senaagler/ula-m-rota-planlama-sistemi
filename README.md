# ula-m-rota-planlama-sistemi
en kısa ,en hızlı rotayı en ucuz şekilde gitmek istenen yere göre  bulmayı harita üzerinden gösteren bir uygulama
import org.json.JSONArray;
import org.json.JSONObject;
import javafx.animation.PauseTransition;
import javafx.scene.image.Image;
import javafx.scene.image.ImageView;
import javafx.util.Duration;


import java.io.OutputStream;
import java.io.PrintStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.time.LocalDate;
import java.time.MonthDay;
import java.util.*;

import javafx.application.Application;
import javafx.application.Platform;
import javafx.geometry.HPos;
import javafx.geometry.Insets;
import javafx.geometry.Pos;
import javafx.scene.Scene;
import javafx.scene.control.*;
import javafx.scene.layout.GridPane;
import javafx.scene.layout.HBox;
import javafx.scene.layout.Priority;
import javafx.scene.layout.VBox;
import javafx.scene.web.WebEngine;
import javafx.scene.web.WebView;
import javafx.stage.Stage;

public class TopluTasimSistemi extends Application {
    private static Map<String, Durak> duraklar = new HashMap<>();
    private static double taksiAcilisUcreti;
    private static double taksiKmBasiUcret;
    private ComboBox<String> filtreCombo;

    private WebEngine webEngine;
    public static void veriYukle(String dosyaAdi) {
        try {
            Path path = Paths.get(dosyaAdi);
            String icerik = new String(Files.readAllBytes(path));
            JSONObject json = new JSONObject(icerik);

           
            JSONArray durakDizisi = json.getJSONArray("duraklar");
            duraklar.clear();

            for (int i = 0; i < durakDizisi.length(); i++) {
                Durak durak = new Durak(durakDizisi.getJSONObject(i));
                duraklar.put(durak.id, durak);
            }

            
            taksiAcilisUcreti = json.getJSONObject("taxi").getDouble("openingFee");
            taksiKmBasiUcret = json.getJSONObject("taxi").getDouble("costPerKm");

            System.out.println("📌 JSON'dan " + duraklar.size() + " durak yüklendi.");
            System.out.println("🚖 Taksi Açılış Ücreti: " + taksiAcilisUcreti + " TL");
            System.out.println("🚖 Taksi Km Başına Ücret: " + taksiKmBasiUcret + " TL");

        } catch (Exception e) {
            System.out.println("❌ JSON yükleme hatası!");
            e.printStackTrace();
        }
    }


   
    

    private void loadAnaEkran(Stage primaryStage) {
        primaryStage.setTitle("🚍 S2GO İzmit Ulaşım Planlayıcı");

        
        GridPane grid = new GridPane();
        grid.setPadding(new Insets(10));
        grid.setVgap(8);
        grid.setHgap(10);

        Label baslangicLatLabel = new Label("📍 Başlangıç Enlem:");
        TextField baslangicLatInput = new TextField();
        grid.add(baslangicLatLabel, 0, 0);
        grid.add(baslangicLatInput, 1, 0);

        Label baslangicLonLabel = new Label("📍 Başlangıç Boylam:");
        TextField baslangicLonInput = new TextField();
        grid.add(baslangicLonLabel, 0, 1);
        grid.add(baslangicLonInput, 1, 1);

        Label hedefLatLabel = new Label("🎯 Hedef Enlem:");
        TextField hedefLatInput = new TextField();
        grid.add(hedefLatLabel, 0, 2);
        grid.add(hedefLatInput, 1, 2);

        Label hedefLonLabel = new Label("🎯 Hedef Boylam:");
        TextField hedefLonInput = new TextField();
        grid.add(hedefLonLabel, 0, 3);
        grid.add(hedefLonInput, 1, 3);

        Label tarihLabel = new Label("📅 Seyahat Tarihi:");
        DatePicker tarihPicker = new DatePicker();
        grid.add(tarihLabel, 0, 4);
        grid.add(tarihPicker, 1, 4);

        Label yolcuLabel = new Label("👤 Yolcu Tipi:");
        ComboBox<String> yolcuCombo = new ComboBox<>();
        yolcuCombo.getItems().addAll("Genel", "Öğrenci", "Yaşlı");
        yolcuCombo.setValue("Genel");
        grid.add(yolcuLabel, 0, 5);
        grid.add(yolcuCombo, 1, 5);

        Label odemeLabel = new Label("💳 Ödeme Yöntemi:");
        ComboBox<String> odemeCombo = new ComboBox<>();
        odemeCombo.getItems().addAll("Nakit", "Kredi Kartı", "Kentkart");
        odemeCombo.setValue("Nakit");
        grid.add(odemeLabel, 0, 6);
        grid.add(odemeCombo, 1, 6);

        Label bakiyeLabel = new Label("💰 Bakiye:");
        TextField bakiyeInput = new TextField();
        grid.add(bakiyeLabel, 0, 7);
        grid.add(bakiyeInput, 1, 7);

        Label krediLabel = new Label("💳 Kredi Kartı Limiti:");
        TextField krediInput = new TextField();
        krediInput.setDisable(true);
        grid.add(krediLabel, 0, 8);
        grid.add(krediInput, 1, 8);

        odemeCombo.setOnAction(e -> {
            boolean krediSecildi = odemeCombo.getValue().equals("Kredi Kartı");
            krediInput.setDisable(!krediSecildi);
            if (!krediSecildi) krediInput.clear();
        });

        Label rotaLabel = new Label("🧭 Alternatif Rotalar:");
        ComboBox<String> rotaCombo = new ComboBox<>();
        rotaCombo.getItems().addAll(
                "Sadece Otobüs",
                "Sadece Tramvay",
                "Otobüs + Tramvay Aktarması",
                "Taksi + Otobüs veya Tramvay"
        );
        rotaCombo.setValue("Otobüs + Tramvay Aktarması");
        grid.add(rotaLabel, 0, 9);
        grid.add(rotaCombo, 1, 9);

        
        Button hesaplaButton = new Button("🚀 En İyi Rotayı Hesapla");
        grid.add(hesaplaButton, 0, 11, 2, 1);
        GridPane.setHalignment(hesaplaButton, HPos.CENTER);

        
        TextArea sonucEkrani = new TextArea();
        sonucEkrani.setEditable(false);
        sonucEkrani.setPrefHeight(180);

        
        CheckBox darkModeToggle = new CheckBox("🌗 Karanlık Mod");
        darkModeToggle.getStyleClass().add("toggle-switch");

        
        VBox solKutu = new VBox(10, grid, sonucEkrani, darkModeToggle);
        solKutu.setPadding(new Insets(10));
        solKutu.setPrefWidth(500);
        VBox.setVgrow(sonucEkrani, Priority.ALWAYS);

        
        WebView haritaGorunumu = new WebView();
        haritaGorunumu.setPrefHeight(700);
        haritaGorunumu.setPrefWidth(800);
        VBox.setVgrow(haritaGorunumu, Priority.ALWAYS);

        webEngine = haritaGorunumu.getEngine();
        webEngine.load(getClass().getResource("map.html").toExternalForm());

        
        veriYukle("C:\\Users\\senag\\eclipse-workspace\\TopluTasimSistemi\\src\\transport_data.json");
        sonucEkrani.appendText("📌 JSON'dan " + duraklar.size() + " durak yüklendi.\n\n");
       
        hesaplaButton.setOnAction(e -> {
            try {
                sonucEkrani.clear();

                double baslat = Double.parseDouble(baslangicLatInput.getText());
                double baslon = Double.parseDouble(baslangicLonInput.getText());
                double hedelat = Double.parseDouble(hedefLatInput.getText());
                double hedelon = Double.parseDouble(hedefLonInput.getText());

                LocalDate tarih = tarihPicker.getValue();
                if (tarih == null) {
                    sonucEkrani.appendText("❌ Lütfen tarih seçin.\n");
                    return;
                }

                Durak basDurak = enYakinDurak(baslat, baslon, sonucEkrani);
                Durak hedDurak = enYakinDurak(hedelat, hedelon, sonucEkrani);

                Yolcu yolcu = switch (yolcuCombo.getValue()) {
                    case "Öğrenci" -> new OgrenciYolcu();
                    case "Yaşlı" -> new YasliYolcu();
                    default -> new GenelYolcu();
                };

               
                if (Yolcu.ozelGunMu(tarih)) {
                    sonucEkrani.appendText("🎉 Bugün özel bir gün! Toplu taşıma ücretsizdir.\n\n");
                }
                double bakiye = bakiyeInput.getText().isEmpty() ? 0.0 : Double.parseDouble(bakiyeInput.getText());
                double krediLimiti = krediInput.getText().isEmpty() ? 0.0 : Double.parseDouble(krediInput.getText());

                Odeme odeme = kullaniciOdemeGirisi(odemeCombo.getValue(), bakiye, krediLimiti);
                String secilenRota = rotaCombo.getValue();

                switch (secilenRota) {
                    case "Sadece Otobüs" ->
                    enIyiRotaBulFiltreli(basDurak.id, hedDurak.id, yolcu, "bus", sonucEkrani, tarih, webEngine);

                    case "Sadece Tramvay" ->
                    enIyiRotaBulFiltreli(basDurak.id, hedDurak.id, yolcu, "bus", sonucEkrani, tarih, webEngine);

                    case "Taksi + Otobüs veya Tramvay" ->
                    enIyiRotaBulTaksiIle(basDurak.id, hedDurak.id, yolcu, sonucEkrani, tarih, webEngine);

                    default ->
                        enIyiRotaBulDijkstra(basDurak.id, hedDurak.id, yolcu, sonucEkrani, webEngine, filtreCombo.getValue(), tarih);
                }

            } catch (Exception ex) {
                sonucEkrani.appendText("❌ Girdi hatası: " + ex.getMessage() + "\n");
            }
        });



        Label filtreLabel = new Label("🔍 Rota Filtresi:");
        filtreCombo = new ComboBox<>();
        filtreCombo.getItems().addAll("En Ucuz", "En Hızlı", "En Az Aktarma");
        filtreCombo.setValue("En Ucuz");
        grid.add(filtreLabel, 0, 10);
        grid.add(filtreCombo, 1, 10);


        darkModeToggle.setOnAction(e -> stilDegistir(grid.getScene(), darkModeToggle.isSelected()));

        
        HBox anaYerlesim = new HBox(20, solKutu, haritaGorunumu);
        anaYerlesim.setPadding(new Insets(10));
        HBox.setHgrow(haritaGorunumu, Priority.ALWAYS);

        
        Scene scene = new Scene(anaYerlesim, 1350, 750);
        scene.getStylesheets().add(getClass().getResource("style.css").toExternalForm());

        primaryStage.setScene(scene);
        primaryStage.setResizable(true);
        primaryStage.setMaximized(true);
        primaryStage.show();
    }


    public void stilDegistir(Scene scene, boolean darkMode) {
        if (darkMode) {
            scene.getRoot().setStyle(
                "-fx-base: #2b2b2b;" +
                "-fx-background-color: #2b2b2b;" +
                "-fx-text-fill: white;" +
                "-fx-control-inner-background: #3c3f41;" +
                "-fx-accent: #4CAF50;" +
                "-fx-focus-color: #4CAF50;"
            );
        } else {
            scene.getRoot().setStyle(""); 
        }
    }


    public static void haritaUzerineCiz(List<RotaSegmenti> rotaSegmentleri, WebEngine webEngine) {
        try {
            JSONArray jsonArray = new JSONArray();

            for (int i = 0; i < rotaSegmentleri.size(); i++) {
                RotaSegmenti seg = rotaSegmentleri.get(i);
                Durak durak = duraklar.get(seg.durakId);
                if (durak != null) {
                    JSONObject nokta = new JSONObject();
                    nokta.put("lat", durak.enlem);
                    nokta.put("lon", durak.boylam);
                    nokta.put("name", durak.ad);
                    nokta.put("mode", seg.mode); 

                    if (i == 0) {
                        nokta.put("type", "start");
                    } else if (i == rotaSegmentleri.size() - 1) {
                        nokta.put("type", "end");
                    } else {
                        nokta.put("type", "normal");
                    }

                    jsonArray.put(nokta);
                }
            }

            String jsKomut = "drawRoute(" + jsonArray.toString() + ")";
            webEngine.executeScript(jsKomut);
            System.out.println("🗺 Haritada mod bazlı çizim yapıldı.");
        } catch (Exception e) {
            System.out.println("❌ Harita çizimi hatası: " + e.getMessage());
            e.printStackTrace();
        }
    }

    
    public void konsoluTextAreaIleDegistir(TextArea hedefEkran) {
        PrintStream guiConsole = new PrintStream(new OutputStream() {
            public void write(int b) {
                Platform.runLater(() -> hedefEkran.appendText(String.valueOf((char) b)));
            }
        });
        System.setOut(guiConsole);
        System.setErr(guiConsole);
    }
 
    public static void enIyiRotaBulFiltreli(
            String baslangicDurakId,
            String hedefDurakId,
            Yolcu yolcu,
            String aracTuru,
            TextArea ekran,
            LocalDate tarih,
            WebEngine webEngine
    ) {
        List<String> rota = new ArrayList<>();
        List<RotaSegmenti> haritaRotasi = new ArrayList<>(); 

        double toplamUcret = 0;
        int toplamSure = 0;
        double toplamMesafe = 0;
        Set<String> ziyaretEdilenDuraklar = new HashSet<>();

        Durak mevcutDurak = duraklar.get(baslangicDurakId);
        ekran.appendText("📍 Başlangıç: " + duraklar.get(baslangicDurakId).ad + "\n");
        ekran.appendText("🌟 Hedef: " + duraklar.get(hedefDurakId).ad + "\n\n");

        if (Yolcu.ozelGunMu(tarih)) {
            ekran.appendText("🎉 Bugün özel bir gün! Toplu taşıma ücretsizdir.\n\n");
        }

        ekran.appendText("🚏 Rota Detayları:\n");

        while (mevcutDurak != null && !mevcutDurak.id.equals(hedefDurakId)) {
            Rota secilenRota = null;

            for (Rota r : mevcutDurak.sonrakiDuraklar) {
                Durak sonrakiDurak = duraklar.get(r.durakId);
                if (sonrakiDurak != null && sonrakiDurak.tur.equals(aracTuru) && !ziyaretEdilenDuraklar.contains(sonrakiDurak.id)) {
                    secilenRota = r;
                    ziyaretEdilenDuraklar.add(sonrakiDurak.id);
                    break;
                }
            }

            if (secilenRota == null) {
                ekran.appendText("❌ Bu ulaşım türüyle hedefe ulaşılamaz!\n");
                return;
            }

            Durak sonrakiDurak = duraklar.get(secilenRota.durakId);
            double ucret = yolcu.indirimUygula(secilenRota.ucret, tarih);
            toplamUcret += ucret;
            toplamSure += secilenRota.sure;
            toplamMesafe += secilenRota.mesafe;

            ekran.appendText("" + rota.size() + "⃣ ");
            ekran.appendText(mevcutDurak.ad + " → " + sonrakiDurak.ad + "\n");
            ekran.appendText("⏳ Süre: " + secilenRota.sure + " dk\n");
            ekran.appendText("💰 Ücret: " + secilenRota.ucret + " TL → " + yolcu.yolcuTipi + ": " + String.format("%.2f", ucret) + " TL\n\n");

            haritaRotasi.add(new RotaSegmenti(mevcutDurak.id, aracTuru));
            haritaRotasi.add(new RotaSegmenti(sonrakiDurak.id, aracTuru));



            mevcutDurak = sonrakiDurak;
        }

        ekran.appendText("📊 Toplam:\n");
        ekran.appendText("💰 " + String.format("%.2f", toplamUcret) + " TL\n");
        ekran.appendText("⏳ " + toplamSure + " dk\n");
        ekran.appendText("📏 " + String.format("%.2f", toplamMesafe) + " km\n\n");

        // 🗺 Harita çizimi
        haritaUzerineCiz(haritaRotasi, webEngine);
    }

    
    public static double aktarmaIndirimiUygula(Durak mevcutDurak, Durak sonrakiDurak, double ucret) {
        if (mevcutDurak.aktarma != null && mevcutDurak.aktarma.aktarmaDurakId.equals(sonrakiDurak.id)) {
            System.out.println("🔄 Aktarma indirimi uygulandı: -" + mevcutDurak.aktarma.aktarmaUcreti + " TL");
            return ucret - mevcutDurak.aktarma.aktarmaUcreti;
        }
        return ucret;
    }
    
    public static void enIyiRotaBulTaksiIle(
            String baslangicDurakId,
            String hedefDurakId,
            Yolcu yolcu,
            TextArea ekran,
            LocalDate tarih,
            WebEngine webEngine 
    ) {
        Durak baslangicDurak = duraklar.get(baslangicDurakId);
        Durak hedefDurak = duraklar.get(hedefDurakId);

        if (baslangicDurak == null || hedefDurak == null) {
            ekran.appendText("❌ Hata: Belirtilen duraklar bulunamadı!\n");
            return;
        }

        double mesafe = MesafeHesaplayici.hesapla(baslangicDurak.enlem, baslangicDurak.boylam, hedefDurak.enlem, hedefDurak.boylam);
        double taksiUcreti = taksiAcilisUcreti + (mesafe * taksiKmBasiUcret);
        double indirimliUcret = yolcu.indirimUygula(taksiUcreti, tarih);

        if (Yolcu.ozelGunMu(tarih)) {
            ekran.appendText("🎉 Bugün özel bir gün! Taksi hariç toplu taşıma ücretsizdir.\n\n");
        }

        ekran.appendText("\n🚖 Alternatif Rota (Taksi):\n");
        ekran.appendText("📏 Mesafe: " + String.format("%.2f", mesafe) + " km\n");
        ekran.appendText("💰 Ücret: " + String.format("%.2f", taksiUcreti) + " TL → " +
                            yolcu.yolcuTipi + " indirimi: " + String.format("%.2f", indirimliUcret) + " TL\n");

        ekran.appendText("\n🚕 Rota:\n");
        ekran.appendText("1️⃣ Taksi: " + baslangicDurak.ad + " → " + hedefDurak.ad + "\n");
        ekran.appendText("⏳ Süre: " + (int)(mesafe * 2) + " dk (yaklaşık)\n");

        ekran.appendText("\n📊 Toplam:\n");
        ekran.appendText("💰 " + String.format("%.2f", indirimliUcret) + " TL\n");
        ekran.appendText("⏳ " + (int)(mesafe * 2) + " dk\n");

        
        List<RotaSegmenti> haritaRotasi = new ArrayList<>();
        haritaRotasi.add(new RotaSegmenti(baslangicDurakId, "taxi"));
        haritaRotasi.add(new RotaSegmenti(hedefDurakId, "taxi"));
        haritaUzerineCiz(haritaRotasi, webEngine);
    }

	private void haritadaGoster(final WebEngine engine, final double lat, final double lon, final String aciklama) {
	    Platform.runLater(() -> {
	        engine.executeScript("addMarker(" + lat + ", " + lon + ", '" + aciklama + "')");
	    });
	}

	private void haritayiTemizle(final WebEngine engine) {
	    Platform.runLater(() -> {
	        engine.executeScript("clearMarkers()");
	    });
	}



	public static Durak enYakinDurak(double kullaniciLat, double kullaniciLon, TextArea logEkrani) {
	    Durak enYakin = null;
	    double minMesafe = Double.MAX_VALUE;

	    for (Durak durak : duraklar.values()) {
	        double mesafe = MesafeHesaplayici.hesapla(kullaniciLat, kullaniciLon, durak.enlem, durak.boylam);
	        if (mesafe < minMesafe) {
	            minMesafe = mesafe;
	            enYakin = durak;
	        }
	    }

	    if (enYakin != null) {
	        logEkrani.appendText("🔍 En Yakın Durak: " + enYakin.ad + " (" + String.format("%.2f", minMesafe) + " km)\n");
	    } else {
	        logEkrani.appendText("❌ En yakın durak bulunamadı!\n");
	    }

	    return enYakin;
	}

	public static void enIyiRotaBulDijkstra(
	        String baslangicDurakId,
	        String hedefDurakId,
	        Yolcu yolcu,
	        TextArea logEkrani,
	        WebEngine webEngine,
	        String filtreTuru,
	        LocalDate tarih
	) {
	    class RotaBilgisi {
	        String durakId;
	        double toplamUcret;
	        int toplamSure;
	        double toplamMesafe;
	        List<String> rotaMetni;
	        List<RotaSegmenti> segmentler;

	        RotaBilgisi(String durakId, double toplamUcret, int toplamSure, double toplamMesafe,
	                    List<String> rotaMetni, List<RotaSegmenti> segmentler) {
	            this.durakId = durakId;
	            this.toplamUcret = toplamUcret;
	            this.toplamSure = toplamSure;
	            this.toplamMesafe = toplamMesafe;
	            this.rotaMetni = new ArrayList<>(rotaMetni);
	            this.segmentler = new ArrayList<>(segmentler);
	        }
	    }

	    Comparator<RotaBilgisi> comparator = switch (filtreTuru) {
	        case "En Hızlı" -> Comparator.comparingInt(rb -> rb.toplamSure);
	        case "En Az Aktarma" -> Comparator.comparingInt(rb -> rb.rotaMetni.size());
	        default -> Comparator.comparingDouble(rb -> rb.toplamUcret);
	    };

	    PriorityQueue<RotaBilgisi> kuyruk = new PriorityQueue<>(comparator);
	    Map<String, Double> enDusukUcret = new HashMap<>();
	    Map<String, Integer> enKisaSure = new HashMap<>();

	    kuyruk.add(new RotaBilgisi(baslangicDurakId, 0, 0, 0.0, new ArrayList<>(), new ArrayList<>()));
	    enDusukUcret.put(baslangicDurakId, 0.0);
	    enKisaSure.put(baslangicDurakId, 0);

	    boolean ozelGun = Yolcu.ozelGunMu(tarih);
	    if (ozelGun) {
	        logEkrani.appendText("🎉 Bugün özel bir gün! Toplu taşıma ücretsizdir.\n\n");
	    }

	    while (!kuyruk.isEmpty()) {
	        RotaBilgisi mevcutRota = kuyruk.poll();
	        String mevcutDurakId = mevcutRota.durakId;

	        if (mevcutDurakId.equals(hedefDurakId)) {
	            logEkrani.appendText("📍 Başlangıç: " + duraklar.get(baslangicDurakId).ad + "\n");
	            logEkrani.appendText("🌟 Hedef: " + duraklar.get(hedefDurakId).ad + "\n\n");
	            logEkrani.appendText("🚏 Rota Detayları:\n");

	            int adim = 1;
	            for (String satir : mevcutRota.rotaMetni) {
	                logEkrani.appendText(adim + "⃣ " + satir + "\n");
	                adim++;
	            }

	            logEkrani.appendText("\n📊 Toplam:\n");
	            logEkrani.appendText("💰 " + String.format("%.2f", mevcutRota.toplamUcret) + " TL\n");
	            logEkrani.appendText("⏳ " + mevcutRota.toplamSure + " dk\n");
	            logEkrani.appendText("📏 " + String.format("%.2f", mevcutRota.toplamMesafe) + " km\n");

	            
	            haritaUzerineCiz(mevcutRota.segmentler, webEngine);
	            return;
	        }

	        Durak mevcutDurak = duraklar.get(mevcutDurakId);
	        if (mevcutDurak == null) continue;

	        for (Rota rota : mevcutDurak.sonrakiDuraklar) {
	            Durak sonrakiDurak = duraklar.get(rota.durakId);
	            if (sonrakiDurak == null) continue;

	            double ucret = yolcu.indirimUygula(rota.ucret, tarih);
	            ucret = aktarmaIndirimiUygula(mevcutDurak, sonrakiDurak, ucret);

	            int yeniSure = mevcutRota.toplamSure + rota.sure;
	            double yeniMesafe = mevcutRota.toplamMesafe + rota.mesafe;
	            double yeniToplamUcret = mevcutRota.toplamUcret + ucret;

	            List<String> yeniMetin = new ArrayList<>(mevcutRota.rotaMetni);
	            String adimMetni = mevcutDurak.ad + " → " + sonrakiDurak.ad +
	                    " (" + mevcutDurak.tur + ") | ⏳ " + rota.sure + " dk | 💰 " +
	                    String.format("%.2f", ucret) + " TL";

	            if (mevcutDurak.aktarma != null && mevcutDurak.aktarma.aktarmaDurakId.equals(sonrakiDurak.id)) {
	                adimMetni += "\n   🔄 Aktarma: +" + mevcutDurak.aktarma.aktarmaSuresi +
	                        " dk, -" + mevcutDurak.aktarma.aktarmaUcreti + " TL";
	                yeniSure += mevcutDurak.aktarma.aktarmaSuresi;
	                yeniToplamUcret -= mevcutDurak.aktarma.aktarmaUcreti;
	            }

	            yeniMetin.add(adimMetni);

	            List<RotaSegmenti> yeniSegmentler = new ArrayList<>(mevcutRota.segmentler);
	            String mode = mevcutDurak.tur.equals("tram") ? "tram" : mevcutDurak.tur.equals("bus") ? "bus" : "taxi";
	            yeniSegmentler.add(new RotaSegmenti(mevcutDurak.id, mode));
	            yeniSegmentler.add(new RotaSegmenti(sonrakiDurak.id, mode));

	            boolean dahaIyi = switch (filtreTuru) {
	                case "En Hızlı" -> yeniSure < enKisaSure.getOrDefault(rota.durakId, Integer.MAX_VALUE);
	                case "En Az Aktarma" -> yeniMetin.size() < enKisaSure.getOrDefault(rota.durakId, Integer.MAX_VALUE);
	                default -> yeniToplamUcret < enDusukUcret.getOrDefault(rota.durakId, Double.MAX_VALUE);
	            };

	            if (dahaIyi) {
	                enKisaSure.put(rota.durakId, yeniSure);
	                enDusukUcret.put(rota.durakId, yeniToplamUcret);
	                kuyruk.add(new RotaBilgisi(rota.durakId, yeniToplamUcret, yeniSure, yeniMesafe, yeniMetin, yeniSegmentler));
	            }
	        }
	    }

	    logEkrani.appendText("❌ Uygun rota bulunamadı!\n");
	}


	private boolean ozelGunMu(LocalDate secilenTarih) {
		
		return false;
	}

	
    public static Odeme kullaniciOdemeGirisi(String secilenOdeme, double bakiye, double krediLimiti) {
        return switch (secilenOdeme) {
            case "Nakit" -> new NakitOdeme(bakiye);
            case "Kredi Kartı" -> new KrediKartiOdeme(bakiye, krediLimiti);
            case "Kentkart" -> new KentkartOdeme(bakiye);
            default -> new NakitOdeme(bakiye);
        };
    }

   
    public static boolean odemeIslemi(Odeme odemeYontemi, double tutar) {
        return odemeYontemi.odemeYap(tutar);
    }
    public static void main(String[] args) {
        launch(args);
    }
    @Override
    public void start(Stage primaryStage) {
        try {
           
            ImageView logo = new ImageView(new Image(getClass().getResourceAsStream("S2GO_KOCAELI_LOGO.png")));
            logo.setFitWidth(200);
            logo.setPreserveRatio(true);

            
            Label baslik = new Label("S2GO - İzmit Ulaşım Planlayıcı");
            baslik.setStyle("-fx-font-size: 24px; -fx-font-weight: bold; -fx-text-fill: #2c3e50;");

            
            Label yukleniyor = new Label("Yükleniyor...");
            yukleniyor.setStyle("-fx-font-size: 14px; -fx-text-fill: #7f8c8d;");

            
            ProgressIndicator yuklemeCubuğu = new ProgressIndicator();
            yuklemeCubuğu.setPrefSize(40, 40);

            VBox kutu = new VBox(20, logo, baslik, yukleniyor, yuklemeCubuğu);
            kutu.setAlignment(Pos.CENTER);
            kutu.setStyle("-fx-background-color: linear-gradient(to bottom, #ffffff, #e0f7fa);");

            Scene splashScene = new Scene(kutu, 600, 400);
            primaryStage.setScene(splashScene);
            primaryStage.setTitle("🚍 S2GO - Başlatılıyor...");
           
            primaryStage.show();

            
            PauseTransition delay = new PauseTransition(Duration.seconds(3));
            delay.setOnFinished(event -> loadAnaEkran(primaryStage));
            delay.play();

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
   




}


   class RotaSegmenti {
       String durakId;
       String mode; 

       public RotaSegmenti(String durakId, String mode) {
           this.durakId = durakId;
           this.mode = mode;
       }
   }
   
   
   

class MesafeHesaplayici {
 private static final double DUNYA_YARI_CAPI_KM = 6371.0; 

 public static double hesapla(double lat1, double lon1, double lat2, double lon2) {
     System.out.println("📏 Mesafe hesaplanıyor: (" + lat1 + ", " + lon1 + ") → (" + lat2 + ", " + lon2 + ")");

     double dLat = Math.toRadians(lat2 - lat1);
     double dLon = Math.toRadians(lon2 - lon1);

     double a = Math.sin(dLat / 2) * Math.sin(dLat / 2)
              + Math.cos(Math.toRadians(lat1)) * Math.cos(Math.toRadians(lat2))
              * Math.sin(dLon / 2) * Math.sin(dLon / 2);

     double c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

     double mesafe = DUNYA_YARI_CAPI_KM * c;

     System.out.println("📏 Hesaplanan mesafe: " + mesafe + " km");

     return mesafe;
 }
}




abstract class Yolcu {
    String yolcuTipi;
    double indirimOrani;
    
    public Yolcu(String yolcuTipi, double indirimOrani) {
        this.yolcuTipi = yolcuTipi;
        this.indirimOrani = indirimOrani;
    }
    
    public double indirimUygula(double ucret, LocalDate tarih) {
        if (ozelGunMu(tarih)) {
            return 0.0;
        }
        return ucret * (1 - indirimOrani);
    }

    public static boolean ozelGunMu(LocalDate tarih) {
        if (tarih == null) return false;

        List<MonthDay> ozelGunler = Arrays.asList(
            MonthDay.of(1, 1),   
            MonthDay.of(4, 23),  
            MonthDay.of(5, 1),   
            MonthDay.of(8, 30), 
            MonthDay.of(10, 29) 
        );

        return ozelGunler.contains(MonthDay.from(tarih));
    }

}


class GenelYolcu extends Yolcu {
    public GenelYolcu() {
        super("Genel", 0.0);
    }
}

class OgrenciYolcu extends Yolcu {
    public OgrenciYolcu() {
        super("Öğrenci", 0.5);
    }
}

class YasliYolcu extends Yolcu {
    public YasliYolcu() {
        super("Yaşlı", 0.3);
    }
}


abstract class Arac {
    String aracTipi;
    
    public Arac(String aracTipi) {
        this.aracTipi = aracTipi;
    }
    
    public abstract double ucretHesapla(double mesafe);
}


class Otobus extends Arac {
    public Otobus() { super("Otobüs"); }
    
    public double ucretHesapla(double mesafe) { return 3.0; }
}

class Tramvay extends Arac {
    public Tramvay() { super("Tramvay"); }
    
    public double ucretHesapla(double mesafe) { return 2.5; }
}

class Taksi extends Arac {
    private double acilisUcreti;
    private double kmBasiUcret;
    
    public Taksi(double acilisUcreti, double kmBasiUcret) {
        super("Taksi");
        this.acilisUcreti = acilisUcreti;
        this.kmBasiUcret = kmBasiUcret;
    }
    
    public double ucretHesapla(double mesafe) {
        return acilisUcreti + (mesafe * kmBasiUcret);
    }
}


abstract class Odeme {
 protected double bakiye;

 public Odeme(double bakiye) {
     this.bakiye = bakiye;
 }

 public abstract boolean odemeYap(double miktar);

 public double getBakiye() {
     return bakiye;
 }
}


class NakitOdeme extends Odeme {
 public NakitOdeme(double bakiye) {
     super(bakiye);
 }

 @Override
 public boolean odemeYap(double miktar) {
     if (bakiye >= miktar) {
         bakiye -= miktar;
         System.out.println("💵 Nakit ödeme başarılı! Kalan bakiye: " + bakiye + " TL");
         return true;
     } else {
         System.out.println("❌ Yetersiz nakit! Ödeme başarısız.");
         return false;
     }
 }
}


class KrediKartiOdeme extends Odeme {
 private double krediLimiti;

 public KrediKartiOdeme(double bakiye, double krediLimiti) {
     super(bakiye);
     this.krediLimiti = krediLimiti;
 }

 @Override
 public boolean odemeYap(double miktar) {
     if (bakiye + krediLimiti >= miktar) {
         bakiye -= miktar;
         System.out.println("💳 Kredi kartı ile ödeme başarılı!");
         return true;
     } else {
         System.out.println("❌ Kredi kartı limiti yetersiz! Ödeme başarısız.");
         return false;
     }
 }
}


class KentkartOdeme extends Odeme {
 public KentkartOdeme(double bakiye) {
     super(bakiye);
 }

 @Override
 public boolean odemeYap(double miktar) {
     if (bakiye >= miktar) {
         bakiye -= miktar;
         System.out.println("🛂 Kentkart ile ödeme başarılı! Kalan bakiye: " + bakiye + " TL");
         return true;
     } else {
         System.out.println("❌ Kentkart bakiyesi yetersiz! Ödeme başarısız.");
         return false;
     }
 }
}



class Durak {
    String id;
    String ad;
    String tur;
    double enlem;
    double boylam;
    boolean sonDurak;
    List<Rota> sonrakiDuraklar;
    Aktarma aktarma;

    public Durak(JSONObject json) {
        this.id = json.getString("id");
        this.ad = json.getString("name");
        this.tur = json.getString("type");
        this.enlem = json.getDouble("lat");
        this.boylam = json.getDouble("lon");
        this.sonDurak = json.getBoolean("sonDurak");
        this.sonrakiDuraklar = new ArrayList<>();
        
        if (json.has("transfer") && !json.isNull("transfer")) {
            this.aktarma = new Aktarma(json.getJSONObject("transfer"));
        }
        
        JSONArray next = json.getJSONArray("nextStops");
        for (int i = 0; i < next.length(); i++) {
            this.sonrakiDuraklar.add(new Rota(next.getJSONObject(i)));
        }
    }
}



class Rota {
    String durakId;
    double mesafe;
    int sure;
    double ucret;

    public Rota(JSONObject json) {
        this.durakId = json.getString("stopId");
        this.mesafe = json.getDouble("mesafe");
        this.sure = json.getInt("sure");
        this.ucret = json.getDouble("ucret");
    }
}


class Aktarma {
    String aktarmaDurakId;
    int aktarmaSuresi;
    double aktarmaUcreti;
    
    public Aktarma(JSONObject json) {
        this.aktarmaDurakId = json.getString("transferStopId");
        this.aktarmaSuresi = json.getInt("transferSure");
        this.aktarmaUcreti = json.getDouble("transferUcret");
    }
    
    
}
