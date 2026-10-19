---
layout: default
title: "Original Source"
description: "The seven Java classes of the CS 360 weight tracking application before any enhancement."
permalink: /artifact/original/
---


# Original Source

The weight tracking application as submitted in CS 360. Seven classes, 546 lines. This is the starting point for all three enhancements.

[Back to the portfolio]({{ site.baseurl }}/)

## Contents

- [AddWeightActivity.java](#addweightactivityjava)
- [DatabaseHelper.java](#databasehelperjava)
- [LoginActivity.java](#loginactivityjava)
- [SettingsActivity.java](#settingsactivityjava)
- [WeightAdapter.java](#weightadapterjava)
- [WeightItem.java](#weightitemjava)
- [WeightListActivity.java](#weightlistactivityjava)

## AddWeightActivity.java

```java
package com.example.cs360project;

import android.Manifest;
import android.annotation.SuppressLint;
import android.content.ContentValues;
import android.content.SharedPreferences;
import android.content.pm.PackageManager;
import android.database.Cursor;
import android.database.sqlite.SQLiteDatabase;
import android.os.Bundle;
import android.telephony.SmsManager;
import android.widget.Button;
import android.widget.EditText;
import android.widget.Toast;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.app.ActivityCompat;
import androidx.core.content.ContextCompat;

public class AddWeightActivity extends AppCompatActivity {

    EditText editWeight, editGoal;
    Button btnSave;
    DatabaseHelper dbHelper;
    private static final int SMS_PERMISSION_CODE = 101;
    private long existingId = -1;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_add_weight);

        dbHelper = new DatabaseHelper(this);
        editWeight = findViewById(R.id.editWeightInput);
        editGoal = findViewById(R.id.editGoalInput);
        btnSave = findViewById(R.id.buttonSaveWeight);

        // Check for incoming data for Update functionality
        if (getIntent().hasExtra("ITEM_ID")) {
            existingId = getIntent().getLongExtra("ITEM_ID", -1);
            String weight = getIntent().getStringExtra("ITEM_WEIGHT");
            editWeight.setText(weight);
            btnSave.setText(R.string.update_weight);
        }

        btnSave.setOnClickListener(v -> {
            String weightText = editWeight.getText().toString();
            String goalText = editGoal.getText().toString();

            if (!weightText.isEmpty()) {
                float currentWeight = Float.parseFloat(weightText);
                
                // Save or Update Weight
                if (existingId == -1) {
                    saveWeight(currentWeight);
                } else {
                    updateWeight(existingId, currentWeight);
                }

                // Update Goal if provided
                if (!goalText.isEmpty()) {
                    saveGoal(Float.parseFloat(goalText));
                }

                // Check for goal and handle SMS notification
                checkGoalAndNotify(currentWeight);
            } else {
                Toast.makeText(this, "Please enter a weight", Toast.LENGTH_SHORT).show();
            }
        });
    }

    private void saveWeight(float weight) {
        SQLiteDatabase db = dbHelper.getWritableDatabase();
        ContentValues values = new ContentValues();
        values.put(DatabaseHelper.COL_WEIGHT_VAL, weight);
        values.put(DatabaseHelper.COL_DATE, new java.util.Date().toString());
        db.insert(DatabaseHelper.TABLE_WEIGHTS, null, values);
    }

    private void updateWeight(long id, float weight) {
        SQLiteDatabase db = dbHelper.getWritableDatabase();
        ContentValues values = new ContentValues();
        values.put(DatabaseHelper.COL_WEIGHT_VAL, weight);
        values.put(DatabaseHelper.COL_DATE, new java.util.Date().toString());
        db.update(DatabaseHelper.TABLE_WEIGHTS, values, DatabaseHelper.COL_ID + "=?", new String[]{String.valueOf(id)});
    }

    private void saveGoal(float goal) {
        SQLiteDatabase db = dbHelper.getWritableDatabase();
        // For simplicity, keep one goal for the user
        db.delete(DatabaseHelper.TABLE_GOAL, null, null);
        ContentValues values = new ContentValues();
        values.put(DatabaseHelper.COL_GOAL_VAL, goal);
        db.insert(DatabaseHelper.TABLE_GOAL, null, values);
    }

    private void checkGoalAndNotify(float currentWeight) {
        SQLiteDatabase db = dbHelper.getReadableDatabase();
        Cursor cursor = db.rawQuery("SELECT * FROM " + DatabaseHelper.TABLE_GOAL, null);

        if (cursor.moveToFirst()) {
            float goalWeight = cursor.getFloat(1);
            if (currentWeight <= goalWeight) {
                // Check if user has opted in for SMS in settings
                SharedPreferences prefs = getSharedPreferences("AppPrefs", MODE_PRIVATE);
                boolean smsEnabled = prefs.getBoolean("sms_enabled", true);

                if (smsEnabled) {
                    if (ContextCompat.checkSelfPermission(this, Manifest.permission.SEND_SMS) == PackageManager.PERMISSION_GRANTED) {
                        sendSMS();
                        finish();
                    } else {
                        // Prompt for permission if not already granted
                        ActivityCompat.requestPermissions(this, new String[]{Manifest.permission.SEND_SMS}, SMS_PERMISSION_CODE);
                    }
                    cursor.close();
                    return; // Wait for result or finish
                }
            }
        }
        cursor.close();
        finish();
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        if (requestCode == SMS_PERMISSION_CODE) {
            SharedPreferences.Editor editor = getSharedPreferences("AppPrefs", MODE_PRIVATE).edit();
            if (grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                // User granted permission
                sendSMS();
                editor.putBoolean("sms_enabled", true);
            } else {
                // User denied permission - Application continues without SMS
                Toast.makeText(this, "SMS Permission Denied. Continuing without notification.", Toast.LENGTH_SHORT).show();
                editor.putBoolean("sms_enabled", false);
            }
            editor.apply();
            finish();
        }
    }

    @SuppressLint("UnlocalizedSms")
    private void sendSMS() {
        try {
            SmsManager smsManager = this.getSystemService(SmsManager.class);
            // Using a placeholder number for testing
            smsManager.sendTextMessage("5555215554", null, "Goal Reached! Congratulations!", null, null);
            Toast.makeText(this, "Goal Notification Sent!", Toast.LENGTH_SHORT).show();
        } catch (Exception e) {
            Toast.makeText(this, "Failed to send SMS", Toast.LENGTH_SHORT).show();
        }
    }
}
```

## DatabaseHelper.java

```java
package com.example.cs360project;

import android.content.ContentValues;
import android.content.Context;
import android.database.Cursor;
import android.database.sqlite.SQLiteDatabase;
import android.database.sqlite.SQLiteOpenHelper;

public class DatabaseHelper extends SQLiteOpenHelper {

    private static final String DATABASE_NAME = "weight_tracker.db";
    private static final int DATABASE_VERSION = 1;

    public static final String TABLE_LOGIN = "login";
    public static final String TABLE_WEIGHTS = "weights";
    public static final String TABLE_GOAL = "goal";

    public static final String COL_ID = "_id";
    public static final String COL_USERNAME = "username";
    public static final String COL_PASSWORD = "password";
    public static final String COL_WEIGHT_VAL = "weight";
    public static final String COL_DATE = "date";
    public static final String COL_GOAL_VAL = "goal_weight";

    public DatabaseHelper(Context context) {
        super(context, DATABASE_NAME, null, DATABASE_VERSION);
    }

    @Override
    public void onCreate(SQLiteDatabase db) {
        String createLogin = "CREATE TABLE " + TABLE_LOGIN + " (" +
                COL_ID + " INTEGER PRIMARY KEY AUTOINCREMENT, " +
                COL_USERNAME + " TEXT, " +
                COL_PASSWORD + " TEXT)";
        db.execSQL(createLogin);

        String createWeights = "CREATE TABLE " + TABLE_WEIGHTS + " (" +
                COL_ID + " INTEGER PRIMARY KEY AUTOINCREMENT, " +
                COL_WEIGHT_VAL + " REAL, " +
                COL_DATE + " TEXT)";
        db.execSQL(createWeights);

        String createGoal = "CREATE TABLE " + TABLE_GOAL + " (" +
                COL_ID + " INTEGER PRIMARY KEY AUTOINCREMENT, " +
                COL_GOAL_VAL + " REAL)";
        db.execSQL(createGoal);
    }

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        db.execSQL("DROP TABLE IF EXISTS " + TABLE_LOGIN);
        db.execSQL("DROP TABLE IF EXISTS " + TABLE_WEIGHTS);
        db.execSQL("DROP TABLE IF EXISTS " + TABLE_GOAL);
        onCreate(db);
    }

    public boolean addUser(String user, String pass) {
        SQLiteDatabase db = this.getWritableDatabase();
        ContentValues values = new ContentValues();
        values.put(COL_USERNAME, user);
        values.put(COL_PASSWORD, pass);
        long result = db.insert(TABLE_LOGIN, null, values);
        return result != -1;
    }

    public boolean checkUser(String user, String pass) {
        SQLiteDatabase db = this.getReadableDatabase();
        Cursor cursor = db.rawQuery("SELECT * FROM " + TABLE_LOGIN + " WHERE " +
                COL_USERNAME + "=? AND " + COL_PASSWORD + "=?", new String[]{user, pass});
        boolean exists = cursor.getCount() > 0;
        cursor.close();
        return exists;
    }

    public void updateWeight(long id, float weight) {
        SQLiteDatabase db = this.getWritableDatabase();
        ContentValues values = new ContentValues();
        values.put(COL_WEIGHT_VAL, weight);
        values.put(COL_DATE, new java.util.Date().toString());

        int result = db.update(TABLE_WEIGHTS, values, COL_ID + "=?", new String[]{String.valueOf(id)});
    }
}
```

## LoginActivity.java

```java
package com.example.cs360project;

import android.content.Intent;
import android.os.Bundle;
import android.widget.Button;
import android.widget.EditText;
import android.widget.Toast;
import androidx.appcompat.app.AppCompatActivity;

public class LoginActivity extends AppCompatActivity {

    DatabaseHelper db;
    EditText editUser, editPass;
    Button btnLogin, btnRegister;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_login);

        db = new DatabaseHelper(this);
        editUser = findViewById(R.id.editUsername);
        editPass = findViewById(R.id.editPassword);
        btnLogin = findViewById(R.id.buttonLogin);
        btnRegister = findViewById(R.id.buttonRegister);

        // Login Logic
        btnLogin.setOnClickListener(v -> {
            String user = editUser.getText().toString();
            String pass = editPass.getText().toString();

            if (db.checkUser(user, pass)) {
                Toast.makeText(this, "Login Successful", Toast.LENGTH_SHORT).show();
                // Navigate to Grid Screen
                Intent intent = new Intent(LoginActivity.this, WeightListActivity.class);
                startActivity(intent);
            } else {
                Toast.makeText(this, "Invalid Credentials", Toast.LENGTH_SHORT).show();
            }
        });

        // Register Logic
        btnRegister.setOnClickListener(v -> {
            String user = editUser.getText().toString();
            String pass = editPass.getText().toString();

            if (user.isEmpty() || pass.isEmpty()) {
                Toast.makeText(this, "Please enter all fields", Toast.LENGTH_SHORT).show();
            } else {
                if (db.addUser(user, pass)) {
                    Toast.makeText(this, "Account Created! Please Login.", Toast.LENGTH_SHORT).show();
                } else {
                    Toast.makeText(this, "Registration Failed", Toast.LENGTH_SHORT).show();
                }
            }
        });
    }
}
```

## SettingsActivity.java

```java
package com.example.cs360project;

import android.Manifest;
import android.content.SharedPreferences;
import android.content.pm.PackageManager;
import android.os.Bundle;
import android.widget.Button;
import android.widget.Switch;
import android.widget.Toast;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.app.ActivityCompat;
import androidx.core.content.ContextCompat;

public class SettingsActivity extends AppCompatActivity {

    private Switch smsSwitch;
    private static final int SMS_PERMISSION_CODE = 101;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.fragment_settings); // Using the existing layout

        smsSwitch = findViewById(R.id.sms_int_switch);
        Button btnDeleteAll = findViewById(R.id.delete_all_data);

        SharedPreferences prefs = getSharedPreferences("AppPrefs", MODE_PRIVATE);
        boolean isSmsEnabled = prefs.getBoolean("sms_enabled", false);
        smsSwitch.setChecked(isSmsEnabled);

        smsSwitch.setOnCheckedChangeListener((buttonView, isChecked) -> {
            if (isChecked) {
                // Request permission when user toggles switch ON
                if (ContextCompat.checkSelfPermission(this, Manifest.permission.SEND_SMS) != PackageManager.PERMISSION_GRANTED) {
                    ActivityCompat.requestPermissions(this, new String[]{Manifest.permission.SEND_SMS}, SMS_PERMISSION_CODE);
                } else {
                    updateSmsPreference(true);
                }
            } else {
                updateSmsPreference(false);
            }
        });

        btnDeleteAll.setOnClickListener(v -> {
            DatabaseHelper dbHelper = new DatabaseHelper(this);
            dbHelper.getWritableDatabase().delete(DatabaseHelper.TABLE_WEIGHTS, null, null);
            Toast.makeText(this, "All weight data deleted", Toast.LENGTH_SHORT).show();
        });
    }

    private void updateSmsPreference(boolean enabled) {
        SharedPreferences.Editor editor = getSharedPreferences("AppPrefs", MODE_PRIVATE).edit();
        editor.putBoolean("sms_enabled", enabled);
        editor.apply();
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        if (requestCode == SMS_PERMISSION_CODE) {
            if (grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                updateSmsPreference(true);
                Toast.makeText(this, "SMS Notifications Enabled", Toast.LENGTH_SHORT).show();
            } else {
                updateSmsPreference(false);
                smsSwitch.setChecked(false);
                Toast.makeText(this, "Permission Denied. SMS disabled.", Toast.LENGTH_SHORT).show();
            }
        }
    }
}
```

## WeightAdapter.java

```java
package com.example.cs360project;

import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.ImageButton;
import android.widget.TextView;
import androidx.annotation.NonNull;
import androidx.recyclerview.widget.RecyclerView;
import java.util.List;

public class WeightAdapter extends RecyclerView.Adapter<WeightAdapter.WeightViewHolder> {

    private List<WeightItem> weightList;
    private OnItemClickListener listener;

    public interface OnItemClickListener {
        void onDeleteClick(int position, long id);
        void onEditClick(WeightItem item);
    }

    public WeightAdapter(List<WeightItem> weightList, OnItemClickListener listener) {
        this.weightList = weightList;
        this.listener = listener;
    }

    @NonNull
    @Override
    public WeightViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.list_item_weight, parent, false);
        return new WeightViewHolder(view);
    }

    @Override
    public void onBindViewHolder(@NonNull WeightViewHolder holder, int position) {
        WeightItem item = weightList.get(position);
        holder.textDate.setText(item.getDate());
        holder.textWeight.setText(item.getWeight());

        holder.btnDelete.setOnClickListener(v -> listener.onDeleteClick(position, item.getId()));

        // Listen for item clicks to trigger update
        holder.itemView.setOnClickListener(v -> listener.onEditClick(item));
    }

    @Override
    public int getItemCount() {
        return weightList.size();
    }

    public static class WeightViewHolder extends RecyclerView.ViewHolder {
        TextView textDate, textWeight;
        ImageButton btnDelete;

        public WeightViewHolder(@NonNull View itemView) {
            super(itemView);
            textDate = itemView.findViewById(R.id.textViewDate);
            textWeight = itemView.findViewById(R.id.textViewWeight);
            btnDelete = itemView.findViewById(R.id.buttonDelete);
        }
    }
}
```

## WeightItem.java

```java
package com.example.cs360project;

public class WeightItem {
    private final long id;
    private final String weight;
    private final String date;

    public WeightItem(long id, String weight, String date) {
        this.id = id;
        this.weight = weight;
        this.date = date;
    }

    public long getId() { return id; }
    public String getWeight() { return weight; }
    public String getDate() { return date; }
}
```

## WeightListActivity.java

```java
package com.example.cs360project;

import android.content.Intent;
import android.database.Cursor;
import android.database.sqlite.SQLiteDatabase;
import android.os.Bundle;
import android.view.Menu;
import android.view.MenuItem;
import androidx.annotation.NonNull;
import androidx.appcompat.app.AppCompatActivity;
import androidx.recyclerview.widget.LinearLayoutManager;
import androidx.recyclerview.widget.RecyclerView;
import com.google.android.material.floatingactionbutton.FloatingActionButton;
import java.util.ArrayList;
import java.util.List;

public class WeightListActivity extends AppCompatActivity {

    RecyclerView recyclerView;
    WeightAdapter adapter;
    List<WeightItem> weightList;
    DatabaseHelper dbHelper;
    FloatingActionButton fabAdd;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_weight_list);

        dbHelper = new DatabaseHelper(this);
        recyclerView = findViewById(R.id.recyclerViewWeights);
        fabAdd = findViewById(R.id.fabAddWeight);

        recyclerView.setLayoutManager(new LinearLayoutManager(this));

        loadData();

        fabAdd.setOnClickListener(v -> {
            Intent intent = new Intent(WeightListActivity.this, AddWeightActivity.class);
            startActivity(intent);
        });
    }

    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        getMenuInflater().inflate(R.menu.weight_menu, menu);
        return true;
    }

    @Override
    public boolean onOptionsItemSelected(@NonNull MenuItem item) {
        if (item.getItemId() == R.id.action_settings) {
            Intent intent = new Intent(this, SettingsActivity.class);
            startActivity(intent);
            return true;
        }
        return super.onOptionsItemSelected(item);
    }

    @Override
    protected void onResume() {
        super.onResume();
        loadData();
    }

    private void loadData() {
        weightList = new ArrayList<>();
        SQLiteDatabase db = dbHelper.getReadableDatabase();
        Cursor cursor = db.rawQuery("SELECT * FROM " + DatabaseHelper.TABLE_WEIGHTS, null);

        if (cursor.moveToFirst()) {
            do {
                long id = cursor.getLong(0);
                float weight = cursor.getFloat(1);
                String date = cursor.getString(2);
                weightList.add(new WeightItem(id, String.valueOf(weight), date));
            } while (cursor.moveToNext());
        }
        cursor.close();

        adapter = new WeightAdapter(weightList, new WeightAdapter.OnItemClickListener() {
            @Override
            public void onDeleteClick(int position, long id) {
                SQLiteDatabase writeDb = dbHelper.getWritableDatabase();
                writeDb.delete(DatabaseHelper.TABLE_WEIGHTS, DatabaseHelper.COL_ID + "=?", new String[]{String.valueOf(id)});
                loadData();
            }

            @Override
            public void onEditClick(WeightItem item) {
                Intent intent = new Intent(WeightListActivity.this, AddWeightActivity.class);
                intent.putExtra("ITEM_ID", item.getId());
                intent.putExtra("ITEM_WEIGHT", item.getWeight());
                startActivity(intent);
            }
        });

        recyclerView.setAdapter(adapter);
    }
}
```
