# Computer-Networks
FTP Client-Server Interaction

- Names: 
Patrick MOMO, id: 5298717, email: momopatrick90@yahoo.fr
Pooja Patil, id 27725310, email: poojapatil12@gmail.com 

- Compiling:
	- g++ -std=c++11 ftpd_udp.cpp -lws2_32 -o server.exe
	- g++ -std=c++11 ftp_udp.cpp -lws2_32 -o client.exe

- Running:
	- server.exe 
	- server.exe q 
		- Running this way will not print logs on screen
	- client.exe
	- follow the instructions

- Communication:
	- Each packet is a string of char of the form [PACKET_TYPE][SEQUENC_NUMBER][CONTENT]
		- The first byte of the header is always the packet type (PACKET_TYPE_HANDSHAKE('h'), PACKET_TYPE_CONTENT('c'), PACKET_TYPE_FINISH('f'), PACKET_TYPE_END('e'), PACKET_TYPE_ACK('a'))
		- The sequence number is a string of size: SEQ_NUM_SIZE
		- Packets have the size BUFFER_SIZE, so the MAX_PACKET_CONTENT size is  BUFFER_SIZE - SEQ_NUM_SIZE - 1
	- Packet type:
		- PACKET_TYPE_END('e') Behaves like an error, if any peer receives this, it stops the communcation by raising an exception
		- PACKET_TYPE_FINISH('f') after sending data (like a who file), this packet is used to signal an end of transmission.
		- PACKET_TYPE_CONTENT('c') content when transmitting file
		- PACKET_TYPE_ACK ('a') acknowledgement
	- 

- Server Threading:
	- Each client has handled by the  ClientRequestHandler class, which runs as a thread.
	- There is a main server loop which waits for client messages and creates handlers when need, and also destroys handlers when transmission is complete.
	- The main server loop also writes incomming client messages to the adequate ClientRequestHandler.

- Renaming in the "PUT" command is done by doing the list command and checking if exists. 
- To signify the last packet, The Finish packet is used. Given that the acknowledgement of the finish packet might be last, the receiver waits a certain
number of timeouts in case the sender resends it.

========================================

- Client: All the code in in: ftp_udp.cpp
	- The client contains one main class FTPClient, the contains all the client code
	- FTPClient::run() 
		+ Starts the client by binding to a socket.
		+ Runs a loop and gets the user input.
		+ invokes all the commands (get, put, list)
	- FTPClient::do_handshake()
		+ Does the 3 way handshake by exchanging packets of type PACKET_TYPE_HANDSHAKE to the server.
	- FTPClient::send_data(vector<char> data)
		+ Generic function to send a vector/array of data to server.
		+ Assumes connection already established.
		+ Uses selective repeate to send the data
			- This uses a window size of WINDOW_SIZE
			- After sending the whole data, a PACKET_TYPE_FINISH is used to signal the client, this is tried a 2 times in case of packet drop.
			- Local variable unacked_seqs contains all the unacked sequence numbers of the selective repeate
				+ Local variable unacked_seqs_timeouts contains the timeout of each of the packets in unacked_seqs
	- FTPClient::send_next_packet(std::vector<char> data,unsigned int &send_base, std::list<int> &unacked_seqs, std::list<clock_t> &unacked_seqs_timeouts)
		+ This sends the next packet (send_base) in the selective repeate.
		+ it adds the packet's sequence to unacked_seqs
			+ This is removed when the packet is acknowledged.
		+ it adds the packet's timeout (now()+PACKET_TIMEOUT_CLOCK) to unacked_seqs_timeouts
		+ it move the send_base by 1
		+ This is used by the send_data(...) as part of the selective repeat algorithm
	- FTPClient::resend_timedout_packets(std::vector<char> data,  std::list<int> &unacked_seqs, std::list<clock_t> &unacked_seqs_timeouts)
		+ Goes through all the unacked_seqs_timeouts and resends all the timedout out packets, then updates the timeout value.
	- FTPClient::get_sequence_num(...)
		+ Utitility function to get the sequence number from a packet.
	- bool FTPClient::acknowledge_packet(std::list<int> &unacked_seqs, std::list<clock_t> &unacked_seqs_timeouts):
		+ Waits for PACKET_TYPE_ACK from the server, if received, checks the sequence number, if the sequence number is in unacked_seqs it is removed 
		with its corresponding timeout.
		+ This is used by the send_data(...) as part of the selective repeat algorithm
		+ Returns true if a packet was acknowledge
	- std::vector<char> FTPClient::receive_data()
		+ Receive data using the selective repeat algorithm
		+ Local variable out_of_order_buffer contains out of order packets
		+ Send acknowledgements for packets received in right order
	- FTPClient::send_packet_with_header(char packet_type, int sequence_number, std::vector<char> data)
		+ Insert the packet type and sequence number then send the raw packet to the server.
	- FTPClient::send_packet_to_server(std::vector<char> data) 
		+ Send a single packet to the server
	- FTPClient::receive_packet_from_server
		+ Receive a single packet from the server without locking, the result is stored in FTPClient::input_buffer
		+ In case there nothing was sent from the peer, input_size = -1
	- FTPClient::get_command(std::string filename)
		+ Executes a "get" command by first using send_data(...) to send the commmand, and then get_data(...) to get the result
		+ stores the result in a file.
		+ checks if file exist locally
	- FTPClient::put_command(std::string filename)
		+ Executes a "put" command by first using send_data(...) to send the commmand, and then put_data(...) to send a file
		+ Checks if file exists by invoking a list_command(...)
	- FTPClient::list_command()
		+ Executes a "list" command by first using send_data(...) to send the commmand, and then get_data(...) to get the result

========================================

- Server: All the code in ftpd_udp.cpp
	- class ClientRequestHandler
		+ Handles a signle client requests
	- ClientRequestHandler::start:
		+ Starts a thread to handle a single client connection
		+ Get the client request (get, put, list) and invokes the adequate method
	- ClientRequestHandler::send_file_data(filename)
		+ Executes the get command, by sending content filename to the client
		+ Uses send_data(...)
	- ClientRequestHandler::send_file_list()
		+ Executes the list command by sending file list to client
		+ Uses send_data(...)
	- ClientRequestHandler::receive_file_data()
		+ Executes the put command by receive file content from client and saving result.
		+ Uses receive_data(...)
	- ClientRequestHandler::send_data(vector<char> data)
		+ Generic function to send a vector/array of data to server.
		+ Assumes connection already established.
		+ Uses selective repeate to send the data
			- This uses a window size of WINDOW_SIZE
			- After sending the whole data, a PACKET_TYPE_FINISH is used to signal the client, this is tried a 2 times in case of packet drop.
			- Local variable unacked_seqs contains all the unacked sequence numbers of the selective repeate
				+ Local variable unacked_seqs_timeouts contains the timeout of each of the packets in unacked_seqs
	- ClientRequestHandler::send_next_packet(std::vector<char> data,unsigned int &send_base, std::list<int> &unacked_seqs, std::list<clock_t> &unacked_seqs_timeouts)
		+ This sends the next packet (send_base) in the selective repeate.
		+ it adds the packet's sequence to unacked_seqs
			+ This is removed when the packet is acknowledged.
		+ it adds the packet's timeout (now()+PACKET_TIMEOUT_CLOCK) to unacked_seqs_timeouts
		+ it move the send_base by 1
		+ This is used by the send_data(...) as part of the selective repeat algorithm
	- ClientRequestHandler::resend_timedout_packets(std::vector<char> data,  std::list<int> &unacked_seqs, std::list<clock_t> &unacked_seqs_timeouts)
		+ Goes through all the unacked_seqs_timeouts and resends all the timedout out packets, then updates the timeout value.
	- ClientRequestHandler::get_sequence_num(...)
		+ Utitility function to get the sequence number from a packet
	- ClientRequestHandler::acknowledge_packet(std::list<int> &unacked_seqs, std::list<clock_t> &unacked_seqs_timeouts):
		+ Waits for PACKET_TYPE_ACK from the server, if received, checks the sequence number, if the sequence number is in unacked_seqs it is removed 
		with its corresponding timeout.
		+ This is used by the send_data(...) as part of the selective repeat algorithm
		+ Returns true if a packet was acknowledge
	- std::vector<char> ClientRequestHandler::receive_data()
		+ Receive data using the selective repeat algorithm
		+ Local variable out_of_order_buffer contains out of order packets
		+ Send acknowledgements for packets received in right order
	- ClientRequestHandler::wait_for_packet(std::vector<char> packet_types)
		+ Wait for packet_types from the server, any other packet is ignored
	- ClientRequestHandler::send_packet_with_header(char packet_type, int sequence_number, std::vector<char> data)
		+ Insert the packet type and sequence number then send the raw packet to the client.
	- ClientRequestHandler::send_packet_to_client(std::vector<char> data) 
		+ Send a single packet to the client
	- ClientRequestHandler::receive_packet_from_client
		+ Receive a single packet from the server without locking, the result is stored in FTPClient::input_buffer
		+ In case there nothing was sent from the peer, input_size = -1
	- ClientRequestHandler::write_to_input_queue(char* buffer, int length)
		+ THis is used the FTPServer to a the buffer of a specific client request handler 

========================================
server:
	- class FTPServer:
		+ This class waits for data on the server port, then creates and managers the ClientRequestHandler.
	- FTPServer::start
		+ Starts the server
		+ Binds to a port.
		+ Listens for data on the port (non blocking0
			- If  data comes in with type PACKET_TYPE_HANDSHAKE, a ClientRequestHandler is created for that client if not
			already created
			- Any other data is sent to the specific ClientRequestHandler using ClientRequestHandler::write_to_input_queue(...)
			- Removes the ClientRequestHandler the are finished ClientRequestHandler::finished==true

	
