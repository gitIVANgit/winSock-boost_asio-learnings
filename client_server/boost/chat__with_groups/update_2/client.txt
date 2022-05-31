#include<nlohmann/json.hpp>
#include<boost/asio.hpp>
#include<iostream>
#include<string>



class Client {
private:
	int idRoom;
    std::string name;
	boost::asio::io_context iocontext;
	boost::asio::ip::tcp::resolver resolver;
	boost::asio::ip::tcp::resolver::results_type endpoint = resolver.resolve("127.0.0.1", "99");
	boost::asio::ip::tcp::socket sock;

public:
	Client():resolver(iocontext), sock(iocontext) {
		boost::asio::connect(sock, endpoint);
		setData();
	}
	void setData() {
		boost::asio::connect(sock, endpoint);
		bool admin;
		admin = adminCheck();

		int option;

		do{

			std::cout << "choose the option: " << "\n\n" << "--> create room = 1\n--> enter room = 2 \n \n your choice :";
			std::cin >> option;
			system("cls");

		} while (option != 1 && option != 2);

		if (option == 1) {

			std::cout << "enter id of room: ";

			int idRoom;
			std::cin >> idRoom;

			nlohmann::json data;

			data["option"] = option;
			data["name"] = this->name;
			data["id"] = idRoom;

			std::string dataSend = data.dump() + "\n";

			boost::asio::write(sock, boost::asio::buffer(dataSend));
			system("cls");
			std::cout << "\nthe room wass created!\nnot pl enter the room\n";
			Sleep(2000);
			system("cls");
			setData();
			
		}
		else {

			std::cout << "enter id of room: ";
			int idRoom;
			std::cin >> idRoom;

			nlohmann::json data;

			data["option"] = option;
			data["name"] = name;
			data["id"] = idRoom;

			std::string dataSend = data.dump() + "\n";

			boost::asio::write(sock, boost::asio::buffer(dataSend));

			system("cls");

			auto a=std::async(&Client::startWriting,this);
			auto aa = std::async(&Client::startReading, this);

		}	
	}
	void startWriting() {
		std::string msg;
		std::getline(std::cin, msg);
		boost::asio::write(sock, boost::asio::buffer((msg+"\n").c_str(),msg.size()+1));
		startWriting();
	}
	void startReading() {
		boost::asio::streambuf streamBuf;
		boost::asio::read_until(sock, streamBuf, "\n");
		std::istream is(&streamBuf);
		std::string line;
		std::getline(is, line);
		std::cout << line << std::endl;
		startReading();
	}
	bool adminCheck() {
		bool admin=false;
		std::string name;
		std::cout << "enter your name: ";
		std::cin >> name;
		this->name = name;
		if (name == "admin") {
			boost::asio::streambuf streamBuf;
			boost::asio::write(sock, boost::asio::buffer(name + "\n"));
			boost::asio::read_until(sock, streamBuf, "\n");
			std::istream is(&streamBuf);
			std::string ok;
			std::getline(is, ok);
			std::cout << ok << std::endl;
			std::cout << "password:";
			std::string pass;
			std::cin >> pass;
			boost::asio::write(sock, boost::asio::buffer(pass + "\n"));
			
			boost::asio::streambuf streamBuf2;
			boost::asio::read_until(sock, streamBuf2, "\n");
			std::istream is2(&streamBuf2);
			std::string answer;
			std::getline(is2, answer);
			system("cls");
			std::cout << "answer: " << answer<<"\n";
			if (answer == "ok") {
				admin = true;
				std::cout << "you are an admin" << std::endl;
				
				return admin;

			}
			else {
				std::cout << "you are not an admin" << std::endl;
				std::cout << "------------------" << std::endl;

				std::cout << "try again!" << std::endl;
				Sleep(2000);
				system("cls");
				adminCheck();
			}
		}
		else {
			system("cls");
			return admin;

		}
		
	}
};
int main() {
	
	Client a;
}
