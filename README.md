# IOS-le-Mobil-Uygulama-Geli-tirme-II-
CTF Komuta Merkezi (Companion App) - Flutter Kaynak Kodu

Aşağıda, CTF platformunun uzaktan yönetimini sağlayan ve iOS/macOS ortamlarında çalışmak üzere Flutter (Dart) ile geliştirilen "Komuta Merkezi" uygulamasının temel kaynak kodu yer almaktadır. Bu yapı; FastAPI arka ucuyla asenkron olarak haberleşerek canlı oyuncu metriklerini (puan ve oyunlaştırma rozetleri) arayüze yansıtmakta ve platformdaki aktif steganografi/kriptografi görevlerini listelemektedir. Aynı zamanda, adli bilişim görevlerini analiz eden otonom çözücü botun (solver bot) mobil istemci üzerinden tetiklenmesi ve yakalanan bayrak (flag) verilerinin ekrana raporlanması süreçlerini yönetmektedir.

Dart
import 'package:flutter/material.dart';
import 'package:http/http.dart' as http;
import 'dart:convert';

void main() {
  runApp(const CTFCompanionApp());
}

class CTFCompanionApp extends StatelessWidget {
  const CTFCompanionApp({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'CTF Komuta Merkezi',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        brightness: Brightness.dark,
        scaffoldBackgroundColor: const Color(0xFF0D1117),
        primaryColor: const Color(0xFF00FF00),
        appBarTheme: const AppBarTheme(
          backgroundColor: Color(0xFF161B22),
          elevation: 0,
        ),
      ),
      home: const DashboardScreen(),
    );
  }
}

class DashboardScreen extends StatefulWidget {
  const DashboardScreen({Key? key}) : super(key: key);

  @override
  State<DashboardScreen> createState() => _DashboardScreenState();
}

class _DashboardScreenState extends State<DashboardScreen> {
  final String apiUrl = "http://127.0.0.1:8000/api";
  
  Map<String, dynamic>? userProfile;
  List<dynamic> activeChallenges = [];
  bool isLoading = true;

  @override
  void initState() {
    super.initState();
    _fetchData();
  }

  Future<void> _fetchData() async {
    setState(() => isLoading = true);
    try {
      final profileRes = await http.get(Uri.parse('$apiUrl/profile/Tharzorg'));
      final challengeRes = await http.get(Uri.parse('$apiUrl/challenges/random?category=crypto'));

      if (profileRes.statusCode == 200 && challengeRes.statusCode == 200) {
        setState(() {
          userProfile = json.decode(profileRes.body);
          activeChallenges = json.decode(challengeRes.body);
          isLoading = false;
        });
      } else {
        _showError("Sunucudan veri alınamadı. Hata Kodu: ${profileRes.statusCode}");
      }
    } catch (e) {
      _showError("Bağlantı Hatası: App Sandbox (Network) iznini kontrol edin.\n$e");
    }
  }

  Future<void> _triggerSolverBot(String challengeId, String type) async {
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text("Otonom Bot ($type) tetiklendi, analiz ediliyor...")),
    );

    try {
      final res = await http.post(
        Uri.parse('$apiUrl/bot/analyze'),
        headers: {"Content-Type": "application/json"},
        body: json.encode({"challenge_id": challengeId, "tool": type}),
      );

      if (res.statusCode == 200) {
        final data = json.decode(res.body);
        _showSuccessDialog("Bayrak Yakalandı!", data['flag']);
      } else {
        _showError("Bot analizi başarısız oldu.");
      }
    } catch (e) {
      _showError("Bot API'sine ulaşılamadı.");
    }
  }

  void _showError(String message) {
    setState(() => isLoading = false);
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(backgroundColor: Colors.redAccent, content: Text(message)),
    );
  }

  void _showSuccessDialog(String title, String flag) {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        backgroundColor: const Color(0xFF161B22),
        title: Text(title, style: const TextStyle(color: Color(0xFF00FF00))),
        content: Text("Bulunan Bayrak:\n$flag", style: const TextStyle(fontSize: 16)),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text("Kapat", style: TextStyle(color: Colors.white)),
          ),
        ],
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text("Komuta Merkezi", style: TextStyle(fontFamily: 'Courier')),
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh, color: Color(0xFF00FF00)),
            onPressed: _fetchData,
          )
        ],
      ),
      body: isLoading
          ? const Center(child: CircularProgressIndicator(color: Color(0xFF00FF00)))
          : Padding(
              padding: const EdgeInsets.all(16.0),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Container(
                    padding: const EdgeInsets.all(16),
                    decoration: BoxDecoration(
                      color: const Color(0xFF161B22),
                      borderRadius: BorderRadius.circular(8),
                      border: Border.all(color: const Color(0xFF30363D)),
                    ),
                    child: Row(
                      mainAxisAlignment: MainAxisAlignment.spaceBetween,
                      children: [
                        Column(
                          crossAxisAlignment: CrossAxisAlignment.start,
                          children: [
                            Text(
                              userProfile?['username'] ?? "Ajan",
                              style: const TextStyle(fontSize: 20, fontWeight: FontWeight.bold, color: Colors.white),
                            ),
                            const SizedBox(height: 4),
                            Text(
                              "Rozet: ${userProfile?['badge'] ?? 'Siber Vatan Savunucusu'}",
                              style: const TextStyle(color: Color(0xFF00FF00)),
                            ),
                          ],
                        ),
                        Column(
                          children: [
                            const Text("Puan", style: TextStyle(color: Colors.grey)),
                            Text(
                              "${userProfile?['score'] ?? 0}",
                              style: const TextStyle(fontSize: 24, fontWeight: FontWeight.bold),
                            ),
                          ],
                        ),
                      ],
                    ),
                  ),
                  const SizedBox(height: 24),
                  const Text(
                    "AKTİF GÖREVLER (FORENSICS & CRYPTO)",
                    style: TextStyle(fontSize: 14, fontWeight: FontWeight.bold, letterSpacing: 1.2, color: Colors.grey),
                  ),
                  const SizedBox(height: 12),
                  Expanded(
                    child: activeChallenges.isEmpty
                        ? const Center(child: Text("Şu an aktif görev yok."))
                        : ListView.builder(
                            itemCount: activeChallenges.length,
                            itemBuilder: (context, index) {
                              final challenge = activeChallenges[index];
                              return Card(
                                color: const Color(0xFF161B22),
                                margin: const EdgeInsets.only(bottom: 12),
                                child: ListTile(
                                  leading: const Icon(Icons.security, color: Color(0xFF00FF00)),
                                  title: Text(challenge['title'] ?? 'Şüpheli Görsel (Steganografi)', style: const TextStyle(color: Colors.white)),
                                  subtitle: Text(challenge['description'] ?? 'LSB tekniği ile gizlenmiş veriyi analiz et.'),
                                  trailing: ElevatedButton(
                                    style: ElevatedButton.styleFrom(
                                      backgroundColor: const Color(0xFF00FF00).withOpacity(0.2),
                                      foregroundColor: const Color(0xFF00FF00),
                                    ),
                                    onPressed: () => _triggerSolverBot(
                                      challenge['id'].toString(), 
                                      challenge['type'] ?? 'zsteg'
                                    ),
                                    child: const Text("Botu Tetikle"),
                                  ),
                                ),
                              );
                            },
                          ),
                  ),
                ],
              ),
            ),
    );
  }
}
