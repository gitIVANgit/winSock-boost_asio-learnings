#include<nlohmann/json.hpp>
#include<iostream>
#include<string>
#include<vector>
#include<boost/asio.hpp>
#include<algorithm>
#include<sstream>

class Client {
public:
	bool admin;
	std::string name;
	int id;
	std::shared_ptr<boost::asio::ip::tcp::socket> sock;

public:
	Client(std::string _name, int _idRoom, std::shared_ptr<boost::asio::ip::tcp::socket>& _sock, bool _admin) :name(_name), id(_idRoom), sock(_sock), admin(_admin) {

	}
}; 

class Room {
public:

	int id;
	int index;
	std::vector<Client>clients;
	void broadcast(std::string &msg) {

		for (Client cl : clients) {

			std::shared_ptr<boost::asio::ip::tcp::socket> sock = cl.sock;

			boost::asio::write(*sock.get(), boost::asio::buffer((msg + "\n").c_str(),msg.size()+1));
		}
			
		
	}
};



class Server {

private:
	std::string pass;
	boost::asio::io_context iocontext;
	boost::asio::ip::tcp::acceptor acceptor;
	std::vector<Room>rooms;
	int indexRoom=-1;
public:

	Server(int i) : acceptor(iocontext, boost::asio::ip::tcp::endpoint(boost::asio::ip::tcp::v4(), 99)) {
		std::cout << "enter password: ";
		std::cin >> pass;
		system("cls");
	}
	
	void run() {

		std::shared_ptr<boost::asio::ip::tcp::socket> sock(new boost::asio::ip::tcp::socket(iocontext));
		acceptor.accept(*sock.get());
		auto a = std::async(&Server::acceptSocket,this,std::ref(sock));
		run();

	}

	void acceptSocket(std::shared_ptr<boost::asio::ip::tcp::socket> &sock) {
		
		int id;
		int option;		
		bool admin=false;

		boost::asio::streambuf streamBuf;

		boost::asio::read_until(*sock.get(), streamBuf, "\n");
		
		std::istream is(&streamBuf);
		std::string line;
		std::getline(is, line);
		std::string name;
		std::string lineForAdmin;

		if (line == "admin") {
			std::cout << "admin tries to connect";
			admin =adminCheck(sock);
			if (!admin) {
				std::cout << "admin failed to log in";
				acceptSocket(sock);

			}else{
				std::cout << "admin logged in";
				boost::asio::streambuf streamBuf2;

				boost::asio::read_until(*sock.get(), streamBuf2, "\n");

				std::istream is2(&streamBuf2);
				std::getline(is2, lineForAdmin);

			}
			
		}
		nlohmann::json j1;
		if (admin) {
			j1 = nlohmann::json::parse(lineForAdmin);
		}
		else {
			 j1 = nlohmann::json::parse(line);
		}
		
		name = j1["name"].get<std::string>();
		option = j1["option"].get<int>();
		id = j1["id"].get<int>();

		if (option == 1) {

			sock->close();

			indexRoom++;
			Room r;
			r.index = indexRoom;
			r.id = id;

			rooms.push_back(r);
			std::cout << " | " << "room added"<< " | "<< r.index << " | " << r.id << " | " <<"\n";
		}
		else {
			for (auto r : rooms) {
				if (r.id == id) {

					rooms[r.index].clients.push_back(Client(name, id, sock,admin));
					auto a = std::async(&Server::startReading, this, std::ref(sock),std::ref(r.index),std::ref(name),std::ref(admin));
					std::cout << "|" << name << " added | " << id << " room id"<< " | " <<r.index << " | ""\n";

					break;
				}
			}
		}
    }
	void startReading(std::shared_ptr<boost::asio::ip::tcp::socket> &sock, int indexRoom,std::string name,bool admin) {
	
			boost::asio::streambuf streamBuf;

			boost::asio::read_until(*sock.get(), streamBuf, "\n");

			std::istream is(&streamBuf);
			std::string line;
			std::getline(is, line);
			if (line == "quit") {
				std::cout << "i want to quit!\n";
				quitClient(name, indexRoom);
				return;
			}
			std::vector<std::string>words;
			std::stringstream ss(line);
			std::string word;
			int count = 0;
			if(admin){
				while (ss >> word) {
					words.push_back(word);
					if (words[0] == "ban" && count == 1) {
						quitClient(words[1], indexRoom);
						startReading(sock, indexRoom, name, admin);	
					 break;
					}
					count++;
				}
			}
			
			line = name +": " + line;

			rooms[indexRoom].broadcast(line);

			startReading(sock, indexRoom,name,admin);
	}
	bool adminCheck(std::shared_ptr<boost::asio::ip::tcp::socket>& sock) {
		
		boost::asio::write(*sock.get(), boost::asio::buffer("ok\n"));
	
		boost::asio::streambuf streamBuf;

		boost::asio::read_until(*sock.get(), streamBuf, "\n");

		std::istream is(&streamBuf);
		std::string line;
		std::getline(is, line);

		std::cout<<line;
		if (line == this->pass) {
			std::cout << "\ntrue\n";
			boost::asio::write(*sock.get(), boost::asio::buffer("ok\n"));
			return true;
		}
		else {
			std::cout << "false\n";

			boost::asio::write(*sock.get(), boost::asio::buffer("not\n"));
			return false;

		}
	}
	void quitClient(std::string name, int idRoom) {
			int count=0;
			Room r = rooms[idRoom];
			for (Client cl : r.clients) {
				if (cl.name == name) {

					std::shared_ptr<boost::asio::ip::tcp::socket> sock = cl.sock;
					sock->close();

					std::cout << " *******************\n";

					std::cout<<"client data:\n";

					std::cout << rooms[idRoom].clients[count].name<<" name \n";
					std::cout << rooms[idRoom].clients[count].id<<"  id room for client\n";

					std::cout << "room data:\n";

					std::cout << rooms[idRoom].id << "  id\n";
					std::cout << rooms[idRoom].index << "indexRoom\n";

					rooms[idRoom].clients.erase(rooms[idRoom].clients.begin() + count);
					std::cout << "\n\n client left\n";
					std::cout << " *******************\n";

					return;
				}
				count++;
			}
		
	}
};

int main() {

	Server a(9);
	a.run();
	return 0;
}