#include<iostream>
#include<vector>
#include<string>
#include<bitset>
#include<map>
#include<unordered_set>
#include <algorithm> 
#include <fstream>
#include <functional> 
using namespace std;

struct dic_entry{
    string index ;
    string binary = "";
    int count = 0;
};
struct comparison{
    bool is_consecutive;
    int mismatches;
    string dic_index;
    string dictionary_binary;
    string binary;
    vector<int> mismatch_places;
};

string int_to_binary_5(int num){
    return bitset<5>(num).to_string();
}
string int_to_binary_4(int num){
    return bitset<4>(num).to_string();
}
string int_to_binary_3(int num){
    return bitset<3>(num).to_string();
}
bool cmp(pair<string, int>& a, pair<string, int>& b){
    return a.second < b.second;
}
vector<pair<string,int>> sort_map(map<string, int>& M){
    vector<pair<string, int>> A;
    for (auto &it : M){
        A.push_back(it);
    }
    sort(A.begin(), A.end(), cmp);
    return A;
}

void create_dictionary(vector<dic_entry>& compressed_dictionary,vector<string> original_binary){
    map<string,int> count;
    for(int i=0;i<original_binary.size();i++)
        count[original_binary[i]]++;
    vector<int> temp_count;
    for(auto item:count)
        temp_count.push_back(item.second);
    unordered_set<int> s(temp_count.begin(), temp_count.end());
    temp_count.assign(s.begin(), s.end());
    sort(temp_count.begin(),temp_count.end());
    reverse(temp_count.begin(),temp_count.end());
    int index = 0;
    for(auto &i:temp_count){
        vector<string> same_frequency;
        dic_entry entry;
        bool insert_possible = true;
        for(auto item_count:count){
            if(i==item_count.second){
                same_frequency.push_back(item_count.first);
            }
        }
        map<string,int> temp_binary_priority;
        for(auto freq_binary:same_frequency){
            for(int j=0;j<original_binary.size();j++){
                if(freq_binary==original_binary[j]){
                    temp_binary_priority[freq_binary] = j;
                    break;
                }
            }
        }
        vector<pair<string,int>> priority = sort_map(temp_binary_priority);
        for(auto i_:priority){
            entry.binary = i_.first;
            entry.count = i;
            entry.index = int_to_binary_4(index);
            for(auto check:compressed_dictionary){
                if(check.index==int_to_binary_4(index) && check.binary==i_.first && check.count==i){
                    insert_possible = false;
                    break;
                }
            }
            if(insert_possible){
                compressed_dictionary.push_back(entry);
                if(index==15)
                    break;
            }
            index++;
        }
        if(index==15)
            break;
    }
}

//compare compressed binaries
pair<pair<int,bool>,vector<int>> compare_binaries_compression(string binary, string dictionary){
    //return number of mismatches, is consecutive and places of mismatches
    int consecutive_check = 0;
    int mismatches = 0;
    bool pattern_consistant  = true;
    vector<int> places;
    for(int i=0;i<binary.size();i++){
        if(binary[i]!=dictionary[i]){
            places.push_back(i);
            mismatches++;
            if(mismatches==1){
                pattern_consistant = true;
            }
            if(pattern_consistant){
                consecutive_check++;
            }
        }
        else{
            pattern_consistant = false;
        }
    }
    if(consecutive_check==mismatches){
        return {{mismatches,true},places};
    }
    return {{mismatches,false},places};
}

//producing mis matches
void produce_mismatches(vector<dic_entry> compressed_dictionary,string binary,vector<comparison>& code_status){
    comparison status_dic_binary;
    for(auto entry:compressed_dictionary){
        string dic_binary = entry.binary;
        pair<pair<int,bool>,vector<int>> mismatches_consecutives = compare_binaries_compression(binary,dic_binary);
        status_dic_binary.dic_index = entry.index;
        status_dic_binary.is_consecutive = mismatches_consecutives.first.second;
        status_dic_binary.mismatches = mismatches_consecutives.first.first;
        status_dic_binary.dictionary_binary = dic_binary;
        status_dic_binary.binary = binary;
        status_dic_binary.mismatch_places = mismatches_consecutives.second;
        code_status.push_back(status_dic_binary);
    }
    // sort(code_status.begin(),code_status.end(),[](comparison a,comparison b){return a.mismatches<b.mismatches;});
}

string generate_mask(string dic_binary, string binary, int start_location){
    string mask = "";
    for(int i=start_location;i<start_location+4;i++){
        int xor_bin = ((dic_binary[i]=='0'?0:1 )^ (binary[i]=='0'?0:1));
        mask+= (xor_bin==0?"0":"1");
    }
    return mask;
}

//main compressing function
string compress(vector<dic_entry> compressed_dictionary,vector<string> original_binary){
    string compressed_code = "";
    int i=0;
    while(i<original_binary.size()){
        string binary = original_binary[i];
        vector<comparison> code_status;
        produce_mismatches(compressed_dictionary,binary,code_status);
        //Applying differnt compression techniques
        string original = "",
        bitmask = "",
        one_consecutive = "",
        two_consecutive = "",
        four_consecutive = "",
        two_not_consecutive = "",
        direct = "",
        rle = "";
        string temp_compressed_entry = "";
        int index = 0;
        for(auto entry:code_status){
            if(entry.mismatches>4 || (entry.mismatches==4 && !entry.is_consecutive)){
                original = ("000"+entry.binary);
                temp_compressed_entry = (index==0)?original:temp_compressed_entry;
            }
            if(entry.mismatches==4 && entry.is_consecutive){
                four_consecutive = ("101"+int_to_binary_5(entry.mismatch_places[0])+entry.dic_index);
                temp_compressed_entry = (index==0)?four_consecutive:temp_compressed_entry;
                temp_compressed_entry = (temp_compressed_entry.size()>four_consecutive.size())?four_consecutive:temp_compressed_entry;
            }
            if(entry.mismatches==2 && entry.is_consecutive){
                two_consecutive = ("100"+int_to_binary_5(entry.mismatch_places[0])+entry.dic_index);
                temp_compressed_entry = (index==0)?two_consecutive:temp_compressed_entry;
                temp_compressed_entry = (temp_compressed_entry.size()>two_consecutive.size())?two_consecutive:temp_compressed_entry;
            }
            if(entry.mismatches==1){
                one_consecutive = ("011"+int_to_binary_5(entry.mismatch_places[0])+entry.dic_index);
                temp_compressed_entry = (index==0)?one_consecutive:temp_compressed_entry;
                temp_compressed_entry = (temp_compressed_entry.size()>one_consecutive.size())?one_consecutive:temp_compressed_entry;
            }
            if(entry.mismatches==2 && !entry.is_consecutive && (entry.mismatch_places[1]-entry.mismatch_places[0])<=3){
                //bitmask
                string mask = generate_mask(entry.dictionary_binary,entry.binary,entry.mismatch_places[0]);
                bitmask = ("010"+int_to_binary_5(entry.mismatch_places[0])+mask+entry.dic_index);
                temp_compressed_entry = (index==0)?bitmask:temp_compressed_entry;
                temp_compressed_entry = (temp_compressed_entry.size()>bitmask.size())?bitmask:temp_compressed_entry;
            }
            if(entry.mismatches==2 && !entry.is_consecutive && (entry.mismatch_places[1]-entry.mismatch_places[0])>3){
                two_not_consecutive = ("110"+int_to_binary_5(entry.mismatch_places[0])+int_to_binary_5(entry.mismatch_places[1])+entry.dic_index);
                temp_compressed_entry = (index==0)?two_not_consecutive:temp_compressed_entry;
                temp_compressed_entry = (temp_compressed_entry.size()>two_not_consecutive.size())?two_not_consecutive:temp_compressed_entry;
            }
            if(entry.mismatches==0){
                direct = ("111"+entry.dic_index);
                temp_compressed_entry = (index==0)?direct:temp_compressed_entry;
                temp_compressed_entry = (temp_compressed_entry.size()>direct.size())?direct:temp_compressed_entry;
                break;
            }
            if(entry.mismatches==3 && ((entry.mismatch_places[2]-entry.mismatch_places[0])<4)){
                //bitmask
                string mask = generate_mask(entry.dictionary_binary,entry.binary,entry.mismatch_places[0]);
                bitmask=("010"+int_to_binary_5(entry.mismatch_places[0])+mask+entry.dic_index);
                temp_compressed_entry = (index==0)?bitmask:temp_compressed_entry;
                temp_compressed_entry = (temp_compressed_entry.size()>bitmask.size())?bitmask:temp_compressed_entry;
            }
            if(entry.mismatches==3 && ((entry.mismatch_places[2]-entry.mismatch_places[0])>=4)){
                original = ("000"+entry.binary);
                temp_compressed_entry = (index==0)?original:temp_compressed_entry;
                temp_compressed_entry = (temp_compressed_entry.size()>original.size())?original:temp_compressed_entry;
            }
            index++;
        }
        int rle_counter = 0;
        while(true){
            if(original_binary[i+1]==binary && rle_counter<8){
                rle_counter++;
                i++;
            }
            else if(original_binary[i+1]!=binary || rle_counter>=8){
                if(rle_counter>0){
                    rle = "001"+int_to_binary_3(rle_counter-1);
                }
                break;
            }
        }
        //adding to the compresed string
        compressed_code+=(temp_compressed_entry+rle);

        i++;

//---------------------------debugging printing---------------------------------
        // cout<<"-----------------------------------------------------"<<endl;
        // for(auto test:code_status){
        //     cout<<" Mismatches: "<<test.mismatches<<" Is consecutive: "<<test.is_consecutive<<" Dic index: "<<test.dic_index<<" Dic binary: "<<test.dictionary_binary<<" Binary: "<<test.binary<<" places : ";
        //     for(auto j:test.mismatch_places){
        //         cout<<j<<" ";
        //     }
        //     cout<<endl;
        // }
        // cout<<" +++++++++++++\n Compressed Binary : "<<temp_compressed_entry<<"\n+++++++++++++\n";
        // cout<<" +++++++++\n  Index of the Binary: "<<i-1<<"\n+++++++++++++\n";
        // cout<<" Original Binary : "<<original<<endl;
        // cout<<" One Consecutive Binary : "<<one_consecutive<<endl;
        // cout<<" Two Consecutive Binary : "<<two_consecutive<<endl;
        // cout<<" Two not Consecutive Binary: "<<two_not_consecutive<<endl;
        // cout<<" Four Consecutive Binary : "<<four_consecutive<<endl;
        // cout<<" Direct Binary : "<<direct<<endl;
        // cout<<" Bitmask Binary : "<<bitmask<<endl;
        // cout<<" RLE Binary : "<<rle<<endl;
        // cout<<"-----------------------------------------------------"<<endl;

    }
 //-----------------------------------------------------------------------------
    return compressed_code;
}

string reverse_bitmask(int location, string bitmask, string dictionary_entry){
    string reconstruct = "";
    int mask_coounter = 0;
    for(int i=0;i<dictionary_entry.size();i++){
        if(location<=i && i<location+4){
            int xor_bin = ((dictionary_entry[i]=='0'?0:1 ) ^ (bitmask[mask_coounter]=='0'?0:1));
            reconstruct += (xor_bin==0?"0":"1");
            mask_coounter++;
        }
        else{
            reconstruct += dictionary_entry[i];
        }  
    }
    return reconstruct;
}

//main decompression function
string decompression(map<string,string> dictionary, string compressed_binary){
    string decompressed_binary = "";
    int pointer = 0;
    string prev_code = "";
    while(pointer<compressed_binary.size()){
        string option = compressed_binary.substr(pointer,3);
        pointer+=3;
        if(option=="000"){
            if(pointer+32>=compressed_binary.size()){
                break;
            }
            decompressed_binary+= compressed_binary.substr(pointer,32);
            prev_code = compressed_binary.substr(pointer,32);
            pointer+=32;
        }
        else if(option=="001"){
            if(pointer+3>=compressed_binary.size()){
                break;
            }
            string rle_count = compressed_binary.substr(pointer,3);
            pointer+=3;
            int rle_count_int = stoi(rle_count, 0, 2);
            for(int i=0;i<rle_count_int+1;i++){
                decompressed_binary+=prev_code;
            }
        }
        else if(option=="010"){
            if(pointer+5+4+4>=compressed_binary.size()){
                break;
            }
            string starting_locarion = compressed_binary.substr(pointer,5);
            pointer+=5;
            string bitmask = compressed_binary.substr(pointer,4);
            pointer+=4;
            string dic_index = compressed_binary.substr(pointer,4);
            pointer+=4;
            prev_code = reverse_bitmask(stoi(starting_locarion,0,2),bitmask,dictionary[dic_index]);
            decompressed_binary+=prev_code;
        }
        else if(option=="011"){
            if(pointer+5+4>=compressed_binary.size()){
                break;
            }
            string mismatch_location = compressed_binary.substr(pointer,5);
            pointer+=5;
            string dic_index = compressed_binary.substr(pointer,4);
            pointer+=4;
            prev_code = "";
            for(int i=0;i<dictionary[dic_index].size();i++){
                if(i==stoi(mismatch_location,0,2)){
                    prev_code += (dictionary[dic_index][i]=='0'?"1":"0");
                }
                else{
                    prev_code += dictionary[dic_index][i];
                }
            }
            decompressed_binary+=prev_code;
        }
        else if(option=="100"){
            if(pointer+5+4>=compressed_binary.size()){
                break;
            }
            string mismatch_location = compressed_binary.substr(pointer,5);
            pointer+=5;
            string dic_index = compressed_binary.substr(pointer,4);
            pointer+=4;
            prev_code = "";
            for(int i=0;i<dictionary[dic_index].size();i++){
                if(i==stoi(mismatch_location,0,2)){
                    prev_code += (dictionary[dic_index][i]=='0'?"1":"0");
                }
                else if(i==stoi(mismatch_location,0,2)+1){
                    prev_code += (dictionary[dic_index][i]=='0'?"1":"0");
                }
                else{
                    prev_code += dictionary[dic_index][i];
                }
            }
            decompressed_binary+=prev_code;
        }
        else if(option=="101"){
            if(pointer+5+4>=compressed_binary.size()){
                break;
            }
            string mismatch_location = compressed_binary.substr(pointer,5);
            pointer+=5;
            string dic_index = compressed_binary.substr(pointer,4);
            pointer+=4;
            prev_code = "";
            for(int i=0;i<dictionary[dic_index].size();i++){
                if(i==stoi(mismatch_location,0,2)){
                    prev_code += (dictionary[dic_index][i]=='0'?"1":"0");
                }
                else if(i==stoi(mismatch_location,0,2)+1){
                    prev_code += (dictionary[dic_index][i]=='0'?"1":"0");
                }
                else if(i==stoi(mismatch_location,0,2)+2){
                    prev_code += (dictionary[dic_index][i]=='0'?"1":"0");
                }
                else if(i==stoi(mismatch_location,0,2)+3){
                    prev_code += (dictionary[dic_index][i]=='0'?"1":"0");
                }
                else{
                    prev_code += dictionary[dic_index][i];
                }
            }
            decompressed_binary+=prev_code;
        }
        else if(option=="110"){
            if(pointer+5+5+4>=compressed_binary.size()){
                break;
            }
            string mismatch_location_1 = compressed_binary.substr(pointer,5);
            pointer+=5;
            string mismatch_location_2 = compressed_binary.substr(pointer,5);
            pointer+=5;
            string dic_index = compressed_binary.substr(pointer,4);
            pointer+=4;
            prev_code = "";
            for(int i=0;i<dictionary[dic_index].size();i++){
                if(i==stoi(mismatch_location_1,0,2)){
                    prev_code += (dictionary[dic_index][i]=='0'?"1":"0");
                }
                else if(i==stoi(mismatch_location_2,0,2)){
                    prev_code += (dictionary[dic_index][i]=='0'?"1":"0");
                }
                else{
                    prev_code += dictionary[dic_index][i];
                }
            }
            decompressed_binary+=prev_code;
        }
        else if(option=="111"){
            if(pointer+4>=compressed_binary.size()){
                break;
            }
            string dic_index = compressed_binary.substr(pointer,4);
            pointer+=4;
            decompressed_binary+=dictionary[dic_index];
            prev_code = dictionary[dic_index];
        }

    }
    return decompressed_binary;
}

//!!!!!!!!!!!!!!!!!!!!!!DRIVER CODE!!!!!!!!!!!!!!!!!!!!!!!!!!!
int main(int argc, char *argv[]){

    vector<string> original_binary;
    vector<string> compressed_binary;

    //++++++++++++++++++++++++++++++++COMPRESSING++++++++++++++++++++++++++++++++
    if(*argv[1]=='1'){
        fstream original_file;
        original_file.open("original.txt",ios::in);
        if(original_file.is_open()){
            string binary;
            while(getline(original_file,binary)){
                original_binary.push_back(binary);
            }
        }
        vector<dic_entry> compressed_dictionary;
        create_dictionary(compressed_dictionary,original_binary);
        string compressed_code = compress(compressed_dictionary,original_binary);
        //Appending zeros to balance
        for(int i=0;i<(compressed_code.size()%32);i++){
            compressed_code+="0";
        }
        //-----------------------OUTPUT-Saving Data--------------------------
        fstream COUT_file;
        COUT_file.open("cout.txt",ios::out);
        if(COUT_file.is_open()){
            for(int i=0;i<compressed_code.size();i++){
                if(i%32==0 && i>0){
                    COUT_file<<"\n";
                    // cout<<"\n";
                }
                COUT_file<<compressed_code[i];
                // cout<<compressed_code[i];
            }
            COUT_file<<"\nxxxx"<<endl;
            // cout<<"\nxxxx"<<endl;
        }
        for(auto step:compressed_dictionary){
            // cout<<step.binary<<" : "<<step.index<<" : "<<step.count<<endl;
            COUT_file<<step.binary<<"\n";
        }
        //-------------------------------------------------------------------

    }
    //+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    //++++++++++++++++++++++++++++++++DECOMPRESSING++++++++++++++++++++++++++++++++
    else if(*argv[1]=='2'){//decompressing
        fstream compressed_file;
        compressed_file.open("compressed.txt",ios::in);
        if(compressed_file.is_open()){
            string compressed_binary = "",binary="";
            map<string,string> dictionary;
            bool change_dic = false;
            int dic_index = 0;
            while(getline(compressed_file,binary)){
                if(binary=="xxxx"){
                    change_dic = true;
                    continue;
                }
                if(!change_dic){
                    compressed_binary+=binary;
                }
                else{
                    dictionary[int_to_binary_4(dic_index)] = binary;
                    dic_index++;
                }
            }
            string decompressed_code = decompression(dictionary,compressed_binary);
            //-----------------------OUTPUT-Saving Data--------------------------
            fstream DOUT_file;
            DOUT_file.open("dout.txt",ios::out);
            if(DOUT_file.is_open()){
                for(int i=0;i<decompressed_code.size();i++){
                    if(i%32==0 && i>0){
                        DOUT_file<<"\n";
                        // cout<<"\n";
                    }
                    DOUT_file<<decompressed_code[i];
                    // cout<<decompressed_code[i];
                }
            }
            DOUT_file<<"\n";
            // cout<<endl;
            //-------------------------------------------------------------------
            }

    }
    //+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

}
