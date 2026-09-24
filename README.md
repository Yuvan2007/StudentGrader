// defines  and imports
#define _POSIX_C_SOURCE 200809L
#define _DEFAULT_SOURCE	
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <math.h>
#include <stdbool.h>

// global functions so qsort comparator can access them 
bool average_sort;
int column_sort;
bool descending = true;
bool include_missing = false;
// main function
typedef struct{
	char * name;
	// to store all scores
	double * scores;
	int num_scores;
	double average;     
	}Student;


// Q sort Comparator
int compare_students(const void *a, const void *b){
     // Casting back to student as we have void pointers
     const Student *sa = (const Student *)a;
     const Student *sb = (const Student *)b;
     // getting values to compare
    double val_a;
    double val_b;
    if(average_sort){
		val_a = sa -> average;
        val_b = sb -> average;
     }
     else{
         val_a = sa -> scores[column_sort];
         val_b = sb -> scores[column_sort];
     }
     //  Handling missing values
     bool a_missing = isnan(val_a);
     bool b_missing = isnan(val_b);
     if(a_missing && b_missing){
         return strcmp(sa->name, sb->name);
     }
     if(a_missing){
         return 1;
     }
     if(b_missing){
         return -1;
     }
     if( val_a == val_b){
         return strcmp(sa->name, sb->name);
     }
     if(val_a > val_b){
         if(descending){
             return -1;
         }
         else{
             return 1;
        }
     }
     if(val_b > val_a){
         if(descending){
             return 1;
         }
         else{
            return -1;
         }
     }
     return 0;
}

// Main Function
int main(int argc, char**argv){
	// initialising default cases
	char * input = "std";
	char * output = "std";
	char * column_string = "Average";
	int column_int = 0;
	int opt = 0;
	//Starting getopt Loop and assigning values according to input
	while((opt = getopt(argc, argv, "i:o:s:rz"))!=-1){
		switch(opt){
			case 'i': input = optarg;break;
			case 'o': output = optarg;break;
			// Using strtol to find if number or string
			case 's':{ 
						char * endptr;
						long value = strtol(optarg, &endptr, 10);
						// endptr stops at element that is not of base 10
						if(*endptr == '\0'){
							column_int = (int)value;
							column_string = NULL;
						}
						else{
							column_string = optarg;
						}
						break;
					}					
			case 'r': descending = false;break;
			case 'z': include_missing = true;break;
			case '?': 
				fprintf(stderr, "Invalid or unrecognized command-line option.\n");
				exit(1);
		}
	}
	// Input and Output File Opening
	FILE * input_file = stdin;
	FILE * output_file = stdout;
	if(strcmp(input, "std") != 0){
		if((input_file = fopen(input,	"r")) == NULL){
			fprintf(stderr, "Unable to open input file\n");
			exit(1);
		}	
	}
    if(strcmp(output, "std") != 0){
        if((output_file = fopen(output,   "w")) == NULL){
            fprintf(stderr, "Unable to open output file\n");
            exit(1);
        }
    }
	// Getting Headers 
	char ** headers = NULL;
	char *lineptr = NULL; 
	int num_headers = 0;
	size_t bufsize = 0;
	if((getline(&lineptr, &bufsize, input_file) == -1)){
		if(strcmp(input, "std") != 0){
			fclose(input_file);
		}
		if(strcmp(output, "std") != 0){
			fclose(output_file);
		}
		fprintf(stderr, "Getline() method failed\n");
		exit(1);
	}
	// using strsep and strdup to get the header into headers array
	char * line = lineptr;
	char * token = NULL;
	int capacity = 4;
	headers = malloc(capacity * sizeof(char*));
	// replacing \n with \0 if present
	char *ptr = strchr(lineptr, '\n');
	if(ptr != NULL){
		*ptr = '\0';	
	}
	while((token = strsep(&line, ",")) != NULL){
		if(num_headers >= capacity){
			capacity *= 2;
		headers = realloc(headers, capacity * sizeof(char*));
		}
		headers[num_headers] = strdup(token);
		num_headers++;	
	} 		 
	// geting all the student scores in array format 
	int st_scores = num_headers - 1;
	int num_students = 0;
	int student_capacity = 4;
	Student *students = malloc(student_capacity * (sizeof(Student)));
	char * st_row = NULL;
	size_t size = 0;
	// getting each row
	while((getline(&st_row, &size, input_file) != -1)){
		char * swap = strchr(st_row, '\n');
		if(swap != NULL){
			*swap = '\0';
		}
		char * linecpy = st_row;
		char * token2;
		// Using fields as a temporary placeholder
		// In the row, fields get the elements
		// We free fields and then again reuse it
		char ** fields = NULL;
		int num_fields = 0;
		int fields_capacity = 4;
		fields = malloc(fields_capacity * sizeof(char*));
		while((token2 = strsep(&linecpy, ","))!=NULL){
			if(num_fields >= fields_capacity){
				fields_capacity *= 2;
				fields = realloc(fields, fields_capacity * (sizeof(char*)));
			}
			fields[num_fields] = strdup(token2);
			num_fields++;
		}
		// converting fields to students
		Student new_student;
		new_student.name = strdup(fields[0]);
		new_student.num_scores = st_scores;
		new_student.scores = malloc(st_scores * sizeof(double));
		// Setting up for finding Average
		double sum = 0;
		int count = 0;
		for(int i = 0; i < st_scores; i++){
			// start from 1st column NOT 0th
			char * score_text = fields[i+1];
			if(strlen(score_text) == 0){
				new_student.scores[i] = NAN;
			}
			else{
				// strod to convert value from string to double
				new_student.scores[i] = strtod(score_text, NULL);
				sum += new_student.scores[i];
				count++;
			}
		}
		if(count > 0){
			new_student.average = sum / count;	
		}
		else{
			new_student.average = NAN;
		}
		// Freeing fields
		for(int i = 0; i < num_fields; i++){
			free(fields[i]);
		}
		free(fields);
		// Finally storing in students array
		if(num_students >= student_capacity){
			student_capacity *= 2;
			students = realloc(students, student_capacity * sizeof(Student));
		}
		students[num_students] = new_student;	
		num_students++;
	}
	// Finding whether to sort by column or by index
	// Trying to find index as in scores array or as average
	if(column_string!=NULL && strcmp(column_string, "Average") == 0){
		average_sort = true;
	}
	// named column(Here also trying to find index in scores array)
	else if(column_string!=NULL){
		average_sort = false;
		int index = -1;
		for(int i =1; i < num_headers; i++){
			if(strcmp(headers[i], column_string) == 0){	
				// scores array doesnt have name
				index = i - 1;
				break;
			}
		}
		if(index == -1){
			if(strcmp(input, "std") != 0){
				fclose(input_file);
			}
			if(strcmp(output, "std") != 0){
				fclose(output_file);
			}
			fprintf(stderr, " Column mentioned is not found in Header\n");
			exit(1);
		}
		column_sort = index;
	}
	
	else{
		if(column_int < 1 || column_int > num_headers){
			if(strcmp(input, "std") != 0){
				fclose(input_file);
			}
			if(strcmp(output, "std") != 0){
				fclose(output_file);
			}
			fprintf(stderr, "Column Index out of range\n");
			exit(1);	
		}
		average_sort = false;
		// One as column int is 1 when headers is 0 in name
		// Another as scores array doesnt have name 
		column_sort = column_int - 2;
	}
	// Filtering values according to z
	int filtered_count = 0;
	// worst case- entire students
	Student *filtered = malloc(num_students * sizeof(Student));
	for(int i =0; i < num_students; i++){
		//Ternery Operator -> condition ? value if true : value if false
		double student_value = (average_sort) ? students[i].average : students[i].scores[column_sort];
		if(include_missing || !isnan(student_value)){
			filtered[filtered_count] = students[i];
			filtered_count++;
		}
	}
	// Applying qsort
	qsort(filtered, filtered_count, sizeof(Student), compare_students);
	// Writing to Output File
	for(int i = 0; i < filtered_count; i++){
		if(isnan(filtered[i].average)){
			fprintf(output_file, "%s score: nan\n", filtered[i].name);
		}
		else{
			fprintf(output_file, "%s score: %.2f\n", filtered[i].name, filtered[i].average);
		}
	}
	//Freeing all heap allocated arrays
	for(int i = 0; i < num_headers; i++){
		free(headers[i]);
	}
	free(headers);
	for(int i = 0; i < num_students; i ++){
		// As we used strdup	
		free(students[i].name);
		free(students[i].scores);
	}
	free(students);
	free(filtered);
	//Closing all files
	if(strcmp(input, "std")!=0){ 
		fclose(input_file);
	}
	if(strcmp(output,"std")!= 0){
		fclose(output_file);
	}
	return 0;
}



		
	
	
	
	
